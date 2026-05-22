# ADR 0002 — Mecanismo IPC predominante para FTGO

**Status**: Accepted
**Fecha**: 22 mayo 2026
**Autor**: David — Equipo de Arquitectura
**Insumos**: [PRD](../prd/PRD.md), [FSD](../fsd/FSD.md), [ADR 0001](0001-estilo-arquitectonico.md), Richardson 2019 cap 3 (IPC), cap 4 (Saga), cap 5 (Outbox).

---

## 1. Contexto

[ADR 0001](0001-estilo-arquitectonico.md) eligió descomposición por business capability con 7 microservicios. Esa decisión deja abierta la pregunta crítica: **¿qué mecanismo de comunicación entre servicios se elige como predominante?** La respuesta condiciona la latencia (NFR-01), la disponibilidad (NFR-02), la tolerancia a fallos externos (NFR-04) y la consistencia de datos (NFR-06). También condiciona la presencia o ausencia de un *event broker* en el diagrama C4 de containers.

El FSD ya anticipa que UC-04 (pago) y UC-06 (cancelación) requieren coordinación entre Order, Billing, Kitchen y Delivery, mientras que UC-01 (toma de pedido) y UC-05 (tracking) tienen requisitos de latencia bajos para el consumidor.

## 2. Restricciones que la decisión debe respetar

| ID | Restricción | Origen |
|---|---|---|
| R-01 | Latencia UX < 200 ms p95 en acciones del consumidor | Brief §A.4 + NFR-01 |
| R-02 | Disponibilidad ≥ 99.9 % de toma de pedidos | Brief §A.4 + NFR-02 |
| R-03 | Tolerancia a fallos externos (Stripe, Maps) | Brief §A.4 + NFR-04 |
| R-04 | Consistencia fuerte intra-aggregate; eventual entre servicios (≤ 2 s p95) | Brief §A.4 + NFR-06 |
| R-05 | Trazabilidad end-to-end con `correlation-id` | Brief §A.4 + NFR-07 |
| R-06 | Stack core Java/Spring Boot (≥ 80 % de servicios) | Brief §A.4 + NFR-08 |
| R-07 | Strangler Fig: el monolito legacy debe poder seguir invocando los servicios nuevos | Brief §A.4 |

## 3. Criterio de decisión

- **Mínimo de 3 opciones reales** que cubran el espectro síncrono / asíncrono / híbrido.
- **Dimensiones de comparación obligatorias**:
  1. Cumplimiento de NFR-01 (latencia) para acciones del consumidor.
  2. Cumplimiento de NFR-02 (disponibilidad) y NFR-04 (tolerancia a fallos externos).
  3. Cumplimiento de NFR-06 (consistencia entre servicios).
  4. Coste operativo y complejidad (presencia de broker, gestión de DLQ, observabilidad).
  5. Compatibilidad con Strangler Fig (el monolito sigue vivo).
  6. Trazabilidad a Richardson cap 3 (IPC), cap 4 (Saga), cap 5 (Outbox).

Cada opción declara impacto explícito en al menos R-01..R-04. La decisión exige consecuencias positivas y negativas.

## 4. Opciones consideradas

### Opción 1 — IPC síncrono REST/HTTP exclusivo

**Descripción**: todos los servicios se invocan mutuamente vía REST/HTTPS con JSON. Ningún broker. Para coordinación multi-servicio (UC-04, UC-06) se hace orquestación síncrona desde el Order Service.

**Pros**

- Modelo mental simple: stack trace lineal, debugging directo.
- Compatible inmediato con Spring Boot (RestTemplate / WebClient) y con el monolito legacy.
- Tooling de observabilidad maduro (OpenTelemetry + HTTP).

**Contras**

- **Disponibilidad multiplicativa**: si cada UC toca 3 servicios al 99.9 %, el flujo agregado cae a ≈ 99.7 % → rompe R-02.
- **Tolerancia a fallos externos limitada**: si Stripe está caído, una llamada síncrona desde Order → Billing → Stripe propaga el timeout al usuario → rompe R-03 / NFR-04.
- Riesgo evidente de **distributed monolith** (cap 2 Richardson, anti-patrón) por acoplamiento temporal.
- No facilita Saga (cap 4): obligaría a orquestación síncrona frágil.

**Impacto en NFRs**: NFR-01 ✓ (cuando todo va bien), NFR-02 ✗, NFR-04 ✗, NFR-06 parcial.

**Veredicto**: descartada como mecanismo único.

---

### Opción 2 — IPC asíncrono exclusivo vía broker (Kafka)

**Descripción**: toda la comunicación entre servicios pasa por **Kafka**: publicación de eventos de dominio, lecturas vía materialised views locales en cada servicio (CQRS).

**Pros**

- Aislamiento de fallos perfecto: si un servicio cae, los productores siguen publicando → cubre R-02.
- Tolerancia a Stripe trivial: `PaymentRequested` se persiste en el log de Kafka y se reintenta → cubre R-03 / NFR-04.
- Coordinación natural por **Saga coreográfica** (cap 4 Richardson).
- Compatible con Strangler Fig si el monolito publica/consume el mismo log.

**Contras**

- **Latencia perceptible por el consumidor degradada**: leer "menú del restaurante" o "estado del pedido" vía materialised view local es rápido, pero **escribir** (confirmar pedido) cruza un round-trip de broker → riesgo en NFR-01.
- Coste de mantener vistas locales actualizadas en cada servicio: amplifica los problemas de consistencia eventual (NFR-06 con ventana ≤ 2 s).
- Curva de aprendizaje del equipo (cap 3 Richardson advierte explícitamente sobre la complejidad).
- Difícil para consultas ad-hoc del back office (SH-04) sin agregadores adicionales.

**Impacto en NFRs**: NFR-01 ✗ (escrituras), NFR-02 ✓✓, NFR-04 ✓✓, NFR-06 parcial.

**Veredicto**: descartada como mecanismo único — penaliza la UX del consumidor.

---

### Opción 3 — IPC **híbrido**: REST síncrono para queries del consumidor + eventos asíncronos para integración entre capabilities (RECOMENDADA)

**Descripción**: dos planos de comunicación, derivados directamente del cap 3 de Richardson:

1. **Plano de lectura/escritura del consumidor (síncrono)**: la app móvil y la web admin hablan con un **API Gateway** que enruta a los servicios apropiados vía REST/HTTPS (JSON). Las **lecturas** se sirven desde el servicio dueño con cache local; las **escrituras** confirman al usuario tras una transacción local en el aggregate (Order, Cart) **antes** de iniciar coordinación inter-servicio.
2. **Plano de integración inter-servicio (asíncrono)**: los servicios publican **eventos de dominio** en **Apache Kafka** (broker). Las Sagas (cap 4) y el patrón **Transactional Outbox** (cap 3 + cap 5) garantizan publicación at-least-once. Los consumidores son idempotentes.

**Para fallos externos** (Stripe, Maps, SendGrid/Twilio) se usa **circuit breaker** + outbox + retry exponencial (cap 3.5 Richardson).

**Pros**

- Cumple NFR-01 (latencia < 200 ms) porque las acciones del consumidor sólo requieren una transacción local del aggregate dueño (Order Service confirma `OrderCreated` y devuelve el `orderId` sin esperar a Billing).
- Cumple NFR-02 / NFR-04: si Stripe cae, la intención queda en el outbox de Billing y se reintenta sin bloquear al consumidor.
- Saga coreográfica entre Order, Billing, Kitchen y Delivery → resuelve UC-04 y UC-06 sin orquestación frágil.
- Compatible con Strangler Fig: el monolito legacy puede publicar/consumir los mismos eventos para sincronizarse mientras dura la migración.
- Trazabilidad clara con `correlation-id` propagado por headers HTTP y por headers de Kafka (NFR-07 / R-05).

**Contras**

- Doble curva: el equipo debe dominar **dos** modelos (REST y eventos). Mitigación: convención unificada de schemas (OpenAPI + JSON Schema en eventos) y documentación de patrones (cap 3).
- Riesgo de duplicar lógica en ambos planos: cada UC debe declarar explícitamente qué pasa síncrono y qué pasa asíncrono (regla de granularidad documentada en el FSD §3).
- Coste operativo del broker: Kafka requiere clúster gestionado (≥ 3 brokers), `schema registry`, monitoreo de lag por consumidor.
- Consistencia eventual entre servicios sigue siendo eventual (NFR-06 con ventana ≤ 2 s p95); UI del consumidor debe diseñarse para tolerar estado en transición ("pedido en confirmación").

**Impacto en NFRs**: NFR-01 ✓, NFR-02 ✓, NFR-04 ✓, NFR-06 ✓ (con ventana declarada), NFR-07 ✓.

---

### Opción 4 — gRPC síncrono + Service Mesh (Istio/Linkerd) sin broker

**Descripción**: comunicación entre servicios vía gRPC con HTTP/2 y un Service Mesh que añade retries, circuit breakers, mTLS y observabilidad transversal. Sin broker.

**Pros**

- Latencia inter-servicio muy baja (binario + HTTP/2).
- Contratos fuertes (protobuf) y observabilidad fuera de la app.

**Contras**

- Sigue siendo **síncrono**: el problema de R-02 / R-03 / NFR-04 persiste (Stripe caído → consumidor afectado).
- Service Mesh introduce un componente operativo grande sin que aún existan equipos especializados (riesgo respecto a BG-04 del BRD).
- Stack core Java/Spring Boot tiene tooling REST más maduro que gRPC (R-06).
- No facilita Saga coreográfica.

**Impacto en NFRs**: NFR-01 ✓✓, NFR-02 parcial, NFR-04 ✗, NFR-06 ✗.

**Veredicto**: viable a futuro como complemento de la Opción 3 si se identifica un cuello de botella síncrono específico, no como reemplazo.

## 5. Decisión

Se elige la **Opción 3 — IPC híbrido: REST/JSON síncrono para el plano del consumidor + Apache Kafka asíncrono para integración inter-servicio**, complementado con **Transactional Outbox** (cap 3 + cap 5) y **Saga coreográfica** (cap 4) para flujos multi-servicio (UC-04 pago, UC-06 cancelación).

**Justificación**

- Es la única opción que satisface simultáneamente R-01 (latencia), R-02 (disponibilidad) y R-03/NFR-04 (tolerancia a Stripe) sin caer en disponibilidad multiplicativa.
- Se alinea con el cap 3 de Richardson, que recomienda explícitamente combinar IPC síncrono para queries de UI con mensajería asíncrona para integración.
- Es compatible con Strangler Fig (R-07): el monolito legacy puede coexistir publicando los mismos eventos.

## 6. Consecuencias

### 6.1 Positivas

- **Resiliencia ante caídas de Stripe**: el outbox garantiza que la intención de cobro nunca se pierde, manteniendo NFR-04 sin afectar al consumidor.
- **Latencia preservada para el consumidor** (NFR-01): la respuesta al confirmar pedido depende únicamente del Order Service.
- **Saga coreográfica** habilita UC-06 (cancelación) y UC-04 (pago) sin orquestador central.
- **Aislamiento de fallos**: una caída de Notifications no detiene Order Taking → cumple NFR-02.
- **Trazabilidad** transversal con `correlation-id` propagado por HTTP headers y Kafka headers (NFR-07).
- Compatible con la decisión del [ADR 0001](0001-estilo-arquitectonico.md).

### 6.2 Negativas

- **Coste operativo del broker**: Kafka requiere clúster gestionado (mínimo 3 brokers + Zookeeper o KRaft + schema registry) y observabilidad de lag.
- **Curva doble**: el equipo debe dominar dos paradigmas; los primeros 6 meses se espera mayor tasa de incidentes de integración.
- **Riesgo de eventos mal modelados**: si los eventos publican datos privados del aggregate, se rompe el encapsulamiento del DDD. Mitigación: revisión de contratos de eventos en PR + schema registry.
- **Consistencia eventual visible al usuario**: el cliente puede ver brevemente estados intermedios (`PENDING_PAYMENT`). La UI debe diseñarse para tolerar esa ventana (≤ 2 s p95).
- **Compromiso con Apache Kafka**: cambiar a otro broker más adelante (RabbitMQ, NATS) tendría coste medio-alto. Mitigación: aislar producers/consumers con una capa Spring Cloud Stream o similar.

## 7. Follow-ups

- **POC obligatoria**: migrar Notifications usando el patrón outbox + Kafka antes de tocar Billing.
- **ADR futuro**: estrategia de **schema evolution** (Avro vs JSON Schema en el registry) y política de compatibilidad backward.
- **ADR futuro**: estrategia de **API gateway** (Spring Cloud Gateway vs Kong vs AWS API Gateway) y BFF para Mobile.
- **ADR futuro**: estrategia de **autenticación y autorización** entre servicios (mTLS interno + tokens de servicio).

## 8. Referencias

- Richardson, C. (2019). *Microservices Patterns*. Manning. Cap 3 (IPC), Cap 4 (Saga), Cap 5 (Domain events + Outbox), Cap 11 (Observability).
- [PRD](../prd/PRD.md), [FSD](../fsd/FSD.md), [ADR 0001](0001-estilo-arquitectonico.md).
- [Brief Anexo A](../../examen.md) §A.4.
