# FSD — FTGO (Food To Go)

**Caso**: Marketplace de delivery de comida (Richardson 2019)
**Versión**: 1.0
**Autor**: David — Análisis funcional
**Fecha**: 22 mayo 2026
**Estado**: Aprobado para alimentar ADRs y diagramas C4
**Insumos**: [PRD](../prd/PRD.md), [Brief Anexo A](../../examen.md) (US-01..03), Richardson 2019 caps 3–5.

> FSD **ligero**: ≥ 5 Casos de Uso con Given/When/Then explícito y trazabilidad obligatoria a (a) una US semilla del brief, (b) una capacidad del PRD o (c) un capítulo del libro.

---

## 1. Introducción

Este FSD formaliza los casos de uso (UCs) que la arquitectura objetivo de FTGO debe soportar para satisfacer el [PRD](../prd/PRD.md) y las 3 user stories semilla del [Brief §A.5](../../examen.md). Cubre el ciclo crítico del marketplace: toma de pedido → pago → asignación de courier → tracking, más una capacidad transversal (cancelación) derivada del cap 4 de Richardson y del PRD.

**Alcance funcional**: 6 UCs (UC-01..UC-06). El alcance excluye explícitamente flujos administrativos (gestión de menús, alta de restaurante, payouts internos), que se documentarán en una segunda iteración del FSD.

## 2. Tabla resumen de UCs

| ID | Título | Actor primario | Capacidad PRD | Origen |
|---|---|---|---|---|
| UC-01 | Tomar pedido desde el menú de un restaurante | Consumidor | Order Taking | US-01 |
| UC-02 | Aceptar o rechazar ticket de pedido | Restaurante | Order Fulfillment / Kitchen | US-02 |
| UC-03 | Asignar pedido a un courier | Courier | Delivery | US-03 |
| UC-04 | Procesar pago del pedido | Consumidor (iniciado por sistema) | Billing & Accounting | Richardson cap 4 + NFR-05 |
| UC-05 | Tracking en tiempo real del pedido | Consumidor | Delivery | NFR-01/02 + N-02 (MRD) |
| UC-06 | Cancelar pedido antes de la entrega | Consumidor | Order Taking | Richardson cap 4 (Saga) + N-04 |

## 3. Regla de granularidad (UC nuevo vs flujo alternativo)

Se crea un **UC nuevo** cuando se cumple ≥ 1 de:

1. El actor primario es distinto (Consumidor vs Restaurante vs Courier).
2. La capacidad de negocio del PRD que ejerce el UC es distinta (Order Taking vs Delivery vs Billing).
3. El **estado terminal** del flujo es distinto (pedido entregado vs pedido cancelado vs pedido pagado).

Si sólo cambian los datos de entrada (p. ej. tipo de cocina seleccionado) o existen pasos intermedios opcionales que no cambian actor ni capacidad, se modela como **flujo alternativo** dentro del UC.

---

## 4. Detalle de Casos de Uso

### UC-01 — Tomar pedido desde el menú de un restaurante

| Campo | Valor |
|---|---|
| Actor primario | Consumidor (SH-01) |
| Capacidad PRD | Order Taking |
| Origen | US-01 (Brief §A.5) |
| NFRs impactados | NFR-01 (latencia), NFR-02 (disponibilidad), NFR-06 (consistencia intra-aggregate) |

**Precondiciones**

- El Consumidor tiene una cuenta válida y al menos una dirección de entrega registrada.
- El Restaurante elegido está activo (abierto en horario).
- El método de pago del consumidor está registrado y verificado por Stripe.

**Flujo principal**

1. El Consumidor abre el menú del Restaurante elegido.
2. Agrega ≥ 1 ítem al carrito.
3. Confirma la dirección de entrega y el método de pago.
4. El sistema valida disponibilidad del restaurante y el stock declarado del ítem.
5. El sistema crea el pedido en estado `PENDING_PAYMENT` y devuelve un número único.

**Flujos alternativos**

- *A1 Restaurante fuera de horario*: el sistema rechaza la confirmación con código `RESTAURANT_CLOSED`.
- *A2 Ítem agotado*: el sistema avisa y permite al consumidor sustituir o eliminar el ítem.
- *A3 Dirección fuera de zona*: el sistema sugiere restaurantes alternativos dentro de la zona.

**Postcondiciones**

- Existe un `Order` con identidad única en estado `PENDING_PAYMENT`.
- Se publica un evento `OrderCreated` en el broker (alimenta UC-04 y UC-02).

**Given / When / Then**

- **Given** el Consumidor tiene un carrito con ≥ 1 ítem y un método de pago válido,
- **When** el Consumidor confirma el pedido,
- **Then** el sistema crea el pedido en estado `PENDING_PAYMENT`, devuelve el número de pedido y publica `OrderCreated`.

---

### UC-02 — Aceptar o rechazar ticket de pedido

| Campo | Valor |
|---|---|
| Actor primario | Restaurante (SH-02) |
| Capacidad PRD | Order Fulfillment / Kitchen |
| Origen | US-02 (Brief §A.5) |
| NFRs impactados | NFR-01 (dashboard responsivo), NFR-02 (disponibilidad), NFR-07 (trazabilidad) |

**Precondiciones**

- Existe un pedido en estado `PAYMENT_AUTHORIZED` originado por UC-01 + UC-04.
- El Restaurante tiene sesión activa en su dashboard.

**Flujo principal**

1. El Restaurante recibe la notificación de un ticket nuevo en el dashboard (push o polling).
2. Revisa los ítems y el tiempo solicitado.
3. Acepta declarando un tiempo estimado de preparación (`ETA_prep`).
4. El sistema actualiza el pedido a `ACCEPTED_BY_RESTAURANT` y notifica al consumidor.

**Flujos alternativos**

- *A1 Rechazo*: el Restaurante rechaza con motivo (`OUT_OF_STOCK`, `OVERLOAD`, `TECHNICAL`). El sistema actualiza el pedido a `REJECTED`, dispara la compensación de UC-04 (refund) y notifica al Consumidor.
- *A2 Sin respuesta en 5 min*: el sistema escala (notificación adicional) y a 10 min auto-rechaza.

**Postcondiciones**

- El pedido tiene un estado terminal de aceptación (`ACCEPTED_BY_RESTAURANT`) o de rechazo (`REJECTED`).
- En caso de rechazo, hay un evento `OrderRejected` que dispara la Saga de cancelación (cap 4 Richardson).

**Given / When / Then**

- **Given** existe un pedido en `PAYMENT_AUTHORIZED` y el Restaurante está activo,
- **When** el Restaurante acepta con un `ETA_prep` válido,
- **Then** el sistema cambia el pedido a `ACCEPTED_BY_RESTAURANT`, notifica al Consumidor y publica `OrderAccepted` con el `ETA_prep`.

---

### UC-03 — Asignar pedido a un courier

| Campo | Valor |
|---|---|
| Actor primario | Courier (SH-03) |
| Capacidad PRD | Delivery |
| Origen | US-03 (Brief §A.5) |
| NFRs impactados | NFR-01 (latencia), NFR-04 (degradación Maps), MM-03 (MRD: < 30 s) |

**Precondiciones**

- Existe un pedido en `ACCEPTED_BY_RESTAURANT` con `ETA_prep` conocido.
- Hay ≥ 1 courier disponible dentro de un radio configurable (p. ej. 3 km) del restaurante.

**Flujo principal**

1. El sistema selecciona el courier más cercano usando geolocalización del Delivery Service.
2. Envía una propuesta de asignación con timeout de 30 s.
3. El Courier acepta dentro del timeout.
4. El sistema asigna el pedido, dispara cálculo de ruta optimizada (Google Maps) y notifica al Restaurante.

**Flujos alternativos**

- *A1 Rechazo del courier*: se reintenta con el siguiente courier elegible.
- *A2 Sin couriers disponibles*: el sistema espera hasta 2 min, expande el radio y reintenta. Si persiste, escala al back office (SH-04).
- *A3 Maps degradado*: el sistema usa ruta heurística (distancia recta) y registra la degradación para observabilidad (NFR-04).

**Postcondiciones**

- El pedido tiene `courierId` asignado y estado `ASSIGNED_TO_COURIER`.
- Se publica `CourierAssigned` que habilita el inicio del tracking (UC-05).

**Given / When / Then**

- **Given** existe un pedido aceptado y un courier disponible a ≤ 3 km,
- **When** el sistema le ofrece la entrega y el courier acepta dentro de los 30 s,
- **Then** el sistema marca el pedido como `ASSIGNED_TO_COURIER` y emite la ruta optimizada al courier.

---

### UC-04 — Procesar pago del pedido

| Campo | Valor |
|---|---|
| Actor primario | Sistema (iniciado por evento `OrderCreated`) — Consumidor como afectado |
| Capacidad PRD | Billing & Accounting |
| Origen | Richardson cap 4 (Saga) + NFR-05 (PCI-DSS) |
| NFRs impactados | NFR-04 (tolerancia a Stripe), NFR-05 (cumplimiento), NFR-06 (eventual consistency) |

**Precondiciones**

- Existe un pedido en `PENDING_PAYMENT`.
- El Consumidor tiene un método de pago tokenizado en Stripe.

**Flujo principal**

1. El Billing Service consume `OrderCreated` desde el broker.
2. Realiza una autorización contra Stripe (no captura).
3. Si la autorización tiene éxito, publica `PaymentAuthorized` y el pedido pasa a `PAYMENT_AUTHORIZED`.
4. Tras `OrderAccepted` por el Restaurante (UC-02), captura el pago y publica `PaymentCaptured`.

**Flujos alternativos**

- *A1 Stripe caído*: la intención de pago se persiste en un *outbox* y se reintenta cada 60 s hasta 60 min con backoff exponencial. Mientras tanto el pedido permanece en `PENDING_PAYMENT` pero el Restaurante **no** recibe ticket hasta `PAYMENT_AUTHORIZED` (mantiene NFR-04 sin romper la consistencia del aggregate, NFR-06).
- *A2 Autorización rechazada*: el pedido pasa a `PAYMENT_FAILED` y se dispara compensación (cancelar `OrderCreated`).
- *A3 Captura tardía* (>15 min desde autorización): se relanza la captura; si vuelve a fallar, escala al back office.

**Postcondiciones**

- El pedido tiene estado coherente entre `Order` y `Billing` (`PAYMENT_CAPTURED` o `PAYMENT_FAILED`).
- Ningún dato de tarjeta se almacena fuera de Stripe (NFR-05).

**Given / When / Then**

- **Given** el Billing Service recibió `OrderCreated` y Stripe está disponible,
- **When** intenta autorizar el monto del pedido,
- **Then** publica `PaymentAuthorized` y el pedido pasa a `PAYMENT_AUTHORIZED`; si Stripe está caído, persiste la intención en el outbox y reintenta sin perder el pedido (NFR-04).

---

### UC-05 — Tracking en tiempo real del pedido

| Campo | Valor |
|---|---|
| Actor primario | Consumidor (SH-01) |
| Capacidad PRD | Delivery |
| Origen | Necesidad N-02 (MRD) + NFR-01/02 |
| NFRs impactados | NFR-01 (latencia), NFR-02 (puede degradar a 99.5 %), NFR-07 (correlation-id) |

**Precondiciones**

- Existe un pedido en estado `ASSIGNED_TO_COURIER` o posterior.
- El Consumidor abre la pantalla de tracking en la app.

**Flujo principal**

1. La app abre un canal de actualización (WebSocket o long-polling) con el Delivery Service.
2. El sistema publica eventos `LocationUpdated` cada 5 s mientras el courier se mueve.
3. La app dibuja la posición del courier y actualiza ETA cada 30 s.
4. Cuando el courier llega al consumidor (geofence ≤ 50 m), el sistema permite confirmar la entrega.

**Flujos alternativos**

- *A1 Pérdida de conexión*: la app reintenta con backoff y muestra "última posición" mientras tanto.
- *A2 Disponibilidad del tracking degradada*: el sistema cae a polling cada 30 s sin romper el flujo (NFR-02 degradado a 99.5 %).

**Postcondiciones**

- Se cierra el canal de tracking al confirmar la entrega.
- El historial de posiciones queda anonimizado y retentivo por máximo 24 h (NFR-05 GDPR).

**Given / When / Then**

- **Given** el pedido está `ASSIGNED_TO_COURIER` y el Consumidor abre tracking,
- **When** el courier publica `LocationUpdated`,
- **Then** la app del Consumidor refleja la posición en ≤ 2 s y actualiza el ETA visible.

---

### UC-06 — Cancelar pedido antes de la entrega

| Campo | Valor |
|---|---|
| Actor primario | Consumidor (SH-01) |
| Capacidad PRD | Order Taking |
| Origen | Richardson cap 4 (Saga) + Necesidad N-04 (MRD) |
| NFRs impactados | NFR-06 (consistency Saga), NFR-04 (refund tolerante a Stripe) |

**Precondiciones**

- Existe un pedido en un estado cancelable: `PENDING_PAYMENT`, `PAYMENT_AUTHORIZED` o `ACCEPTED_BY_RESTAURANT` (no se permite cancelar si ya hay courier asignado más allá del radio del restaurante).

**Flujo principal**

1. El Consumidor selecciona "Cancelar pedido" en la app.
2. La app envía la solicitud al Order Service.
3. El Order Service publica `OrderCancellationRequested`.
4. Se ejecuta una **Saga** que orquesta:
   - Refund vía Billing (compensación de UC-04).
   - Liberación del ticket en el Restaurante (compensación de UC-02 si aplica).
5. Cuando todos los pasos compensatorios confirman, el pedido queda en `CANCELLED`.

**Flujos alternativos**

- *A1 Cancelación tardía* (cocina ya comenzó): el sistema aplica política de penalización configurable; si el Consumidor confirma, sigue el flujo principal.
- *A2 Refund falla en Stripe*: la Saga reintenta vía outbox (NFR-04) y escala al back office a los 60 min.

**Postcondiciones**

- Estado final del pedido: `CANCELLED`.
- Todos los efectos colaterales (cobro, ticket, asignación) están deshechos o registrados como pendientes con su `correlation-id` (NFR-07).

**Given / When / Then**

- **Given** existe un pedido en un estado cancelable y el Consumidor solicita cancelar,
- **When** la Saga ejecuta refund y liberación del ticket exitosamente,
- **Then** el pedido queda en `CANCELLED` y el Consumidor recibe confirmación con el `correlation-id` de la operación.

---

## 5. Mapa UC → capacidades PRD

| UC | Capacidad PRD | Confirma cobertura |
|---|---|---|
| UC-01 | Order Taking | ✓ |
| UC-02 | Order Fulfillment / Kitchen | ✓ |
| UC-03 | Delivery | ✓ |
| UC-04 | Billing & Accounting | ✓ |
| UC-05 | Delivery (sub-capacidad tracking) | ✓ |
| UC-06 | Order Taking (compensación Saga) | ✓ |

Cobertura faltante (deferida): **Consumer Management**, **Restaurant Management** y **Notifications** se ejercen indirectamente (sign-up, gestión de menús, push) y se documentarán en una v1.1 del FSD.

## 6. Invariantes funcionales

- Un pedido **no** puede saltar estados (ej. de `PENDING_PAYMENT` directo a `ASSIGNED_TO_COURIER`).
- Ningún UC introduce nuevos stakeholders fuera del PRD §2.
- Cada UC porta `correlation-id` end-to-end (NFR-07).
- La cancelación (UC-06) siempre se ejecuta como Saga; nunca como un `DELETE` atómico.

## 7. Referencias

- [PRD](../prd/PRD.md), [MRD](../mrd/MRD.md), [BRD](../brd/BRD.md).
- Richardson, C. (2019). *Microservices Patterns*. Manning. Cap 3 (IPC), Cap 4 (Saga), Cap 5 (Business Logic), Cap 11 (Observability).
- [Brief Anexo A](../../examen.md) §A.5 (US-01..03).
