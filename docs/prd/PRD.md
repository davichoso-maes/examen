# PRD — FTGO (Food To Go)

**Caso**: Marketplace de delivery de comida (Richardson 2019)
**Versión**: 1.0
**Autor**: David — Equipo de Arquitectura
**Fecha**: 22 mayo 2026
**Estado**: Aprobado para derivar FSD y ADRs
**Fuentes**: [BRD](../brd/BRD.md), [MRD](../mrd/MRD.md), [Anexo A del examen](../../examen.md), Richardson 2019 (caps 1–2).

> PRD **ligero** (2–4 páginas equivalentes) en línea con la consigna. Cubre 5 secciones obligatorias: contexto, stakeholders, capacidades de negocio, NFRs y alcance. Cada NFR cita su origen en el brief.

---

## 1. Contexto y objetivos

FTGO es una plataforma operativa de delivery de comida que conecta consumidores, restaurantes y couriers. El producto vive hoy en un **monolito Java/WAR** con los síntomas clásicos del *infierno monolítico* descritos por Richardson (cap 1): builds lentos, despliegues riesgosos, escalado conflictivo y falta de aislamiento de fallos.

La dirección decidió migrar a **microservicios** mediante **Strangler Fig** durante 18–24 meses ([Brief §A.4 Migración incremental](../../examen.md)). Este PRD documenta los **requisitos de producto** que la nueva arquitectura debe satisfacer: capacidades, NFRs medibles y alcance de la primera versión migrada. Es el insumo directo del [FSD](../fsd/FSD.md) y de los dos [ADRs](../adr/) arquitectónicos.

**Objetivo del producto**: sostener el crecimiento operacional sin degradar la UX del consumidor ni la confiabilidad operativa del restaurante y del courier, habilitando equipos independientes de desarrollo y despliegue (objetivos del [BRD](../brd/BRD.md) BG-01..BG-05).

## 2. Stakeholders

Lista canónica del [Brief §A.2](../../examen.md). No se inventan stakeholders fuera de esta lista.

| ID | Rol | Necesidad principal | Origen |
|---|---|---|---|
| SH-01 | Consumidor | UX rápida, transparencia, tracking en tiempo real | Brief §A.2 |
| SH-02 | Restaurante | Gestión predecible de tickets y carga de cocina | Brief §A.2 |
| SH-03 | Courier | Asignaciones cercanas, ruta optimizada, pago confiable | Brief §A.2 |
| SH-04 | Empleado FTGO (back office) | Visibilidad, reportes, resolución de incidentes | Brief §A.2 |
| SH-05 | Equipo de Arquitectura | Calidad arquitectónica, trazabilidad, mantenibilidad | Brief §A.2 |
| SH-06 | Sistemas externos (Stripe, Google Maps, SendGrid/Twilio) | SLAs estables | Brief §A.2 |

## 3. Capacidades de negocio

Las **7 capacidades** del [Brief §A.3](../../examen.md), alineadas con el cap 2 de Richardson (*Decompose by Business Capability*). Estas capacidades son **candidatas** a microservicios; la granularidad final se decide en [ADR 0001](../adr/0001-estilo-arquitectonico.md).

1. **Consumer Management** — Registro, perfiles, direcciones, preferencias del consumidor. Es la capacidad que sostiene el ciclo de vida del usuario final y habilita personalización y cumplimiento GDPR.
2. **Restaurant Management** — Restaurantes registrados, menús, horarios, disponibilidad. Cambios de menú deben propagarse a las apps sin downtime.
3. **Order Taking** — Toma de pedidos: validación, cálculo de total, confirmación. Es la capacidad transaccional crítica del marketplace y la primera con la que el consumidor mide la calidad del producto.
4. **Order Fulfillment / Kitchen** — Tickets entregados al restaurante, estado de preparación. Su dashboard debe reflejar cambios casi en tiempo real para evitar saturación de cocina (US-02).
5. **Delivery** — Asignación de couriers, rutas y tracking. Depende fuertemente de Google Maps y requiere baja latencia para responder a la app del courier (US-03).
6. **Billing & Accounting** — Cobros, comisiones, payouts a restaurantes y couriers. Debe seguir operando aunque la pasarela externa esté degradada (cola de retry, [Brief §A.4]).
7. **Notifications** — Emails, SMS, push: confirmaciones, alertas, recibos. Asíncrona por naturaleza, aislable del flujo crítico de pedido.

## 4. Requisitos no funcionales (NFRs)

≥ 5 NFRs con métrica numérica y origen explícito.

### NFR-01 — Latencia UX

- **Métrica**: ≤ 200 ms p95 en acciones del consumidor en la app (listar menú, ver carrito, confirmar pedido).
- **Origen**: [Brief §A.4 Latencia UX](../../examen.md).
- **Justificación**: experiencia móvil en horarios pico; abandono de carrito crece con latencia (MM-01 del MRD).

### NFR-02 — Disponibilidad

- **Métrica**: ≥ 99.9 % mensual en el flujo de toma de pedidos (UC-01, UC-04). Tracking en tiempo real (UC-05) puede degradar hasta 99.5 %.
- **Origen**: [Brief §A.4 Disponibilidad](../../examen.md).
- **Justificación**: cada minuto de outage en horario pico tiene impacto directo en ingresos (BRD BG-02).

### NFR-03 — Escalabilidad horizontal independiente

- **Métrica**: el sistema debe soportar **5x** del tráfico base en horarios pico (12–14 h y 19–22 h), escalando cada capacidad de forma independiente (Y-axis del Scale Cube de Richardson cap 1) sin necesidad de escalar las 7 capacidades juntas.
- **Origen**: [Brief §A.4 Carga + Escalabilidad horizontal](../../examen.md).
- **Justificación**: el monolito hoy escala todo o nada y desperdicia recursos.

### NFR-04 — Tolerancia a fallos de sistemas externos

- **Métrica**: el sistema **debe** seguir aceptando pedidos cuando la pasarela de pago (Stripe) está caída, encolando reintentos hasta 60 min. Puede degradar mapas a respuestas en caché por hasta 30 min.
- **Origen**: [Brief §A.4 Tolerancia a fallos externos](../../examen.md).
- **Justificación**: brief explícito + cap 4–5 Richardson (Saga / outbox / circuit breaker).

### NFR-05 — Cumplimiento (PCI-DSS + GDPR)

- **Métrica**: 0 datos de tarjeta persistidos en la plataforma; 100 % del tratamiento de pago delegado a Stripe. Datos personales del consumidor segregados en Consumer Service con retención y consentimiento auditables.
- **Origen**: [Brief §A.4 Cumplimiento](../../examen.md).
- **Justificación**: requisito legal y restricción de auditoría del BRD (BG-05).

### NFR-06 — Consistencia de datos

- **Métrica**: consistencia **fuerte dentro del aggregate de un pedido**; eventual entre servicios para reporting. Ventana de eventual consistency ≤ 2 s p95 entre Order, Delivery y Billing.
- **Origen**: [Brief §A.4 Consistencia de datos](../../examen.md).
- **Justificación**: cap 5–6 Richardson (DB-per-service + Saga).

### NFR-07 — Trazabilidad

- **Métrica**: 100 % de las acciones del consumidor portan `correlation-id` propagado end-to-end; tracing distribuido cubriendo todos los servicios y BDs.
- **Origen**: [Brief §A.4 Trazabilidad](../../examen.md).
- **Justificación**: imprescindible en arquitectura distribuida para soporte (SH-04) y operaciones (cap 11 Richardson).

### NFR-08 — Stack tecnológico del core

- **Métrica**: ≥ 80 % de los servicios core escritos en Java/Spring Boot; libertad tecnológica permitida en servicios satélite (notifications, dashboards internos).
- **Origen**: [Brief §A.4 Tecnología](../../examen.md).
- **Justificación**: reutilización del expertise existente del equipo y reducción del riesgo de migración.

## 5. Alcance

### 5.1 Dentro de alcance (release inicial post-migración)

- Las **7 capacidades** del §3 expuestas como servicios cuyo grado de descomposición se define en [ADR 0001](../adr/0001-estilo-arquitectonico.md).
- UI gateway que enruta progresivamente tráfico **desde el monolito legacy hacia los nuevos servicios** (Strangler Fig).
- Mecanismo IPC predominante decidido en [ADR 0002](../adr/0002-mecanismo-ipc.md), combinando síncrono REST/HTTP para lecturas y asíncrono via broker para integración.
- Observabilidad mínima viable: logs centralizados, métricas RED, tracing con `correlation-id`.
- Integraciones externas: Stripe (pago), Google Maps (geocoding + rutas), SendGrid/Twilio (notificaciones).

### 5.2 Fuera de alcance

- **Big-bang rewrite** del monolito (explícitamente vetado por [Brief §A.4 Migración incremental](../../examen.md)).
- Construcción de un *Backend for Frontend* específico para web admin más allá del gateway compartido.
- Service Mesh completo (Istio/Linkerd): se difiere a un ADR posterior si la operación lo justifica.
- Expansión geográfica y nuevos verticales (cubierto en futuras versiones del MRD).
- Sistemas internos de RRHH / contabilidad corporativa: el `Billing & Accounting` se limita al flujo del marketplace.

### 5.3 Asunciones que condicionan el alcance

- El monolito legacy permanece operativo durante 18–24 meses con acceso de lectura/escritura a sus tablas, expuesto como `System_Ext` en los diagramas C4.
- El equipo de plataforma habilita Kubernetes + observabilidad básica para soportar la operación de los nuevos servicios.
- Stripe y Google Maps mantienen los SLAs vigentes; los circuit breakers se calibran sobre esa base.

## 6. Trazabilidad (recapitulación)

| Elemento del PRD | Origen | Aterriza en |
|---|---|---|
| Capacidades §3 (7 ítems) | Brief §A.3 / Richardson cap 2 | ADR 0001, C4 Container |
| NFR-01 | Brief §A.4 Latencia UX | ADR 0002 (IPC síncrono eficiente), FSD UC-01 |
| NFR-02 | Brief §A.4 Disponibilidad | ADR 0001 (aislamiento por servicio), FSD UC-01..05 |
| NFR-03 | Brief §A.4 Carga + Escalabilidad | ADR 0001 (Y-axis Scale Cube), C4 Container |
| NFR-04 | Brief §A.4 Tolerancia a fallos | ADR 0002 (broker + retry), FSD UC-04 |
| NFR-05 | Brief §A.4 Cumplimiento | FSD UC-04 (pago) |
| NFR-06 | Brief §A.4 Consistencia | ADR 0001 / 0002 (DB-per-service, Saga) |
| NFR-07 | Brief §A.4 Trazabilidad | Observabilidad transversal |
| NFR-08 | Brief §A.4 Tecnología | ADR 0001 / C4 Container (Java/Spring Boot) |

## 7. Anexos y referencias

- Richardson, C. (2019). *Microservices Patterns*. Manning. Capítulos 1–2 (obligatorios), 4–6, 11 (referenciados).
- [Brief FTGO (Anexo A)](../../examen.md).
- Repositorio canónico: <https://github.com/microservices-patterns/ftgo-application>.
- [BRD](../brd/BRD.md) y [MRD](../mrd/MRD.md) como upstream del producto.
