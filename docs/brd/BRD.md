# BRD — Business Requirements Document

**Caso**: FTGO (Food To Go) — migración monolito → microservicios
**Versión**: 1.0
**Autor**: David (Equipo de Arquitectura)
**Fecha**: 21 mayo 2026
**Estado**: Aprobado para iniciar PRD

> Documento de **nivel de negocio**. Define el *por qué* de la inversión arquitectónica. El **MRD** detalla el contexto de mercado y el **PRD** traduce este documento a requisitos de producto y NFRs verificables.

---

## 1. Resumen ejecutivo

FTGO opera desde hace varios años una plataforma de delivery de comida con un **monolito Java/WAR** que ya muestra los síntomas del *infierno monolítico* descritos por Richardson (cap 1): builds lentos, despliegues riesgosos, escalado conflictivo entre módulos, falta de aislamiento de fallos y lock-in tecnológico [Richardson 2019 cap 1].

La dirección decide **migrar a microservicios de forma incremental** (Strangler Fig, 18–24 meses) para sostener el crecimiento sin un *big-bang rewrite*. Este BRD justifica la inversión, fija los objetivos de negocio y delimita el alcance estratégico que alimentará al PRD/FSD/ADRs.

## 2. Objetivos de negocio

| ID | Objetivo | KPI medible | Meta a 24 meses |
|---|---|---|---|
| BG-01 | Sostener el crecimiento de pedidos sin degradación en horarios pico | p95 latencia checkout en horario pico | ≤ 200 ms |
| BG-02 | Reducir el riesgo de caídas globales por fallos locales | % de outages totales atribuibles a 1 sólo módulo | ≤ 10 % |
| BG-03 | Acelerar el time-to-market de nuevas capacidades | Lead time de cambio (commit → prod) por capacidad | ≤ 1 día (de ≈ 1 semana actual) |
| BG-04 | Habilitar contratación y onboarding de equipos independientes | # de equipos que despliegan sin bloquearse entre sí | ≥ 5 |
| BG-05 | Asegurar continuidad legal y regulatoria | Cumplimiento PCI-DSS y GDPR auditado | 100 % en módulos de pago y datos de consumidor |

## 3. Caso de inversión

- **Costo evitado**: pérdidas estimadas por horas-pico-caídas y por *deploys congelados* en viernes (operación actual sufre rollback semanal ≈ 1 vez).
- **Costo asumido**: infraestructura adicional (broker, observabilidad, gateway), formación del equipo en patrones de microservicios, mayor complejidad operativa inicial.
- **Trade-off aceptado por la dirección**: aumento controlado de complejidad operativa a cambio de **escalabilidad independiente** y **resiliencia** (cap 1 Richardson).

## 4. Stakeholders de negocio

| Stakeholder | Interés primario | Origen |
|---|---|---|
| Consumidor | UX rápida, transparencia y tracking en tiempo real | Brief §A.2 |
| Restaurante | Gestión predecible de tickets de cocina | Brief §A.2 |
| Courier | Asignaciones cercanas, pagos confiables | Brief §A.2 |
| Empleado FTGO (back office) | Visibilidad, soporte e incidencias | Brief §A.2 |
| Dirección FTGO | ROI, crecimiento sin riesgo operativo | Inferido del caso (cap 1) |
| Equipo de Arquitectura | Calidad, mantenibilidad y trazabilidad | Brief §A.2 |
| Sistemas externos (Stripe, Google Maps, SendGrid/Twilio) | SLAs predecibles | Brief §A.2 |

## 5. Alcance estratégico (in / out)

**Dentro de alcance**
- Documentar la arquitectura **objetivo** post-migración.
- Definir las 7 capacidades de negocio canónicas (cap 2 Richardson) que serán microservicios candidatos.
- Definir patrón de migración: **Strangler Fig** durante 18–24 meses [Brief §A.4].
- Establecer el marco de NFRs (latencia, disponibilidad, escalabilidad horizontal, tolerancia a fallos externos) [Brief §A.4].

**Fuera de alcance (en este BRD)**
- Big-bang rewrite del monolito.
- Cambio de stack base: el core permanece Java/Spring Boot [Brief §A.4].
- Construcción de nuevos canales (kioscos físicos, voice ordering, etc.).
- Expansión geográfica fuera del mercado actual (eso vive en el MRD).

## 6. Restricciones del negocio

| ID | Restricción | Origen |
|---|---|---|
| BR-01 | Migración debe ser **incremental** (Strangler Fig) | Brief §A.4 Migración incremental |
| BR-02 | Stack core preferido Java/Spring Boot por reutilización del equipo | Brief §A.4 Tecnología |
| BR-03 | PCI-DSS delegado a Stripe; GDPR para datos de consumidor | Brief §A.4 Cumplimiento |
| BR-04 | Disponibilidad mensual de toma de pedidos **≥ 99.9 %** | Brief §A.4 Disponibilidad |
| BR-05 | Sistema debe seguir tomando pedidos aunque la pasarela de pago caiga | Brief §A.4 Tolerancia a fallos externos |

## 7. Supuestos y riesgos

| Tipo | Descripción | Mitigación propuesta |
|---|---|---|
| Supuesto | El equipo absorbe la curva de microservicios en 6 meses | Formación + ADRs claros + pares con consultoría |
| Supuesto | Stripe y Google Maps mantienen sus SLAs históricos | Circuit breakers y degradación graceful (NFR-04) |
| Riesgo | *Distributed monolith* si la descomposición se hace por capas y no por capabilities | ADR 0001 prioriza decomposición por business capability (cap 2 Richardson) |
| Riesgo | Operación incrementalmente más compleja durante la migración | Observabilidad y tracing distribuido obligatorios desde día 1 (NFR-07) |
| Riesgo | Vendor lock-in con el broker elegido | ADR 0002 evalúa alternativas (Kafka vs RabbitMQ vs AWS SNS/SQS) |

## 8. Criterios de éxito del BRD

Este documento se considera completo y útil cuando:

- [x] Cada objetivo de negocio tiene KPI medible y meta numérica.
- [x] Cada restricción se rastrea al Brief §A.4 o al libro Richardson.
- [x] El alcance distingue explícitamente lo que entra y lo que sale.
- [x] Los riesgos están enlazados a una decisión arquitectónica posterior (ADR 0001 / 0002) o a un NFR del PRD.

## 9. Trazabilidad downstream

| Elemento del BRD | Aterriza en |
|---|---|
| BG-01 (latencia pico) | PRD §NFR-01, ADR 0002 (IPC) |
| BG-02 (aislamiento de fallos) | PRD §NFR-02, ADR 0001 (estilo) |
| BG-03 (lead time) | PRD §NFR-03 (escalabilidad organizativa / X-axis) |
| BG-05 (cumplimiento) | PRD §NFR-05, FSD UC-04 (pago) |
| BR-01 (Strangler Fig) | ADR 0001 (decisión y consecuencias) |
| BR-05 (pedidos sin pago en vivo) | FSD UC-04 + ADR 0002 (async / cola de retry) |

## 10. Referencias

- Richardson, C. (2019). *Microservices Patterns*. Manning. Capítulos 1–2.
- [Brief FTGO](../brief/brief.md), §A.1–A.4.
- Repositorio canónico: <https://github.com/microservices-patterns/ftgo-application>.
