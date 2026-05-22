# ADR 0001 — Estilo arquitectónico de la plataforma FTGO

**Status**: Accepted
**Fecha**: 22 mayo 2026
**Autor**: David — Equipo de Arquitectura
**Insumos**: [PRD](../prd/PRD.md), [FSD](../fsd/FSD.md), [Brief FTGO](../brief/brief.md), Richardson 2019 caps 1–2.

---

## 1. Contexto

FTGO opera un **monolito Java/WAR** con los síntomas del *infierno monolítico* descritos por Richardson (cap 1): builds lentos, despliegues frágiles, escalado conflictivo y falta de aislamiento de fallos. La dirección decidió migrar a microservicios con **Strangler Fig** durante 18–24 meses ([Brief §A.4](../brief/brief.md)).

Antes de decidir IPC, persistencia o coreografía, hay que fijar el **estilo arquitectónico** global: ¿quedarse como monolito modular, migrar a un mid-point (modular/cellular), descomponer por capacidades de negocio, o por subdominios DDD? La decisión condiciona los 6 UCs del FSD, la granularidad operativa y el coste de la migración.

## 2. Restricciones que la decisión debe respetar

Lista concreta de NFRs del [PRD §4](../prd/PRD.md) y restricciones del brief que **toda** opción debe satisfacer (no son trade-offs negociables):

| ID | Restricción | Origen |
|---|---|---|
| R-01 | Escalabilidad horizontal independiente — tráfico **5x** en pico | Brief §A.4 Carga + NFR-03 |
| R-02 | Disponibilidad ≥ 99.9 % en toma de pedidos | Brief §A.4 Disponibilidad + NFR-02 |
| R-03 | Latencia < 200 ms p95 en acciones del consumidor | Brief §A.4 Latencia UX + NFR-01 |
| R-04 | Tolerancia a fallos externos (Stripe / Maps) | Brief §A.4 Tolerancia + NFR-04 |
| R-05 | Migración **incremental** Strangler Fig (18–24 meses) | Brief §A.4 Migración incremental |
| R-06 | Stack core preferentemente Java/Spring Boot | Brief §A.4 Tecnología + NFR-08 |
| R-07 | Consistencia fuerte intra-aggregate; eventual entre servicios | Brief §A.4 Consistencia + NFR-06 |
| R-08 | Cumplimiento PCI-DSS + GDPR | Brief §A.4 Cumplimiento + NFR-05 |

## 3. Criterio de decisión

Antes de evaluar opciones se fija explícitamente el método (TODO 2 del prompt semilla):

- **Mínimo de 3 opciones reales** (no de paja).
- **Dimensiones de comparación obligatorias**:
  1. Cobertura de NFR-01 (latencia) y NFR-02 (disponibilidad).
  2. Cobertura de NFR-03 (escalabilidad independiente).
  3. Coste de operación (infra + observabilidad + equipo).
  4. Riesgo de *distributed monolith* (cap 2 Richardson).
  5. Coste de migración respetando R-05 (Strangler Fig).
  6. Trazabilidad al cap 1–2 de Richardson.

Cada opción declara impacto **explícito** en al menos los NFRs R-01 a R-04. La decisión final exige consecuencias positivas **y** negativas.

## 4. Opciones consideradas

### Opción 1 — Mantener monolito modular (status quo + modularización interna)

**Descripción**: refactorizar el monolito en módulos Java fuertemente aislados (paquetes, JPMS) con base de datos compartida. No se introducen microservicios.

**Pros**

- Operación simple, una sola pipeline de despliegue.
- Sin overhead de red, naturalmente cumple NFR-01 (latencia local).
- Riesgo cero de *distributed monolith*.

**Contras**

- **No** habilita escalado independiente: ante el pico 5x sigue siendo *todo o nada* (rompe R-01 / NFR-03).
- Mantiene el lock-in tecnológico y los builds lentos (no resuelve el problema raíz del brief / cap 1).
- Disponibilidad sigue acoplada: una falla en Notifications puede tumbar Order Taking (rompe R-02 / NFR-02).

**Impacto en NFRs**: NFR-01 ✓, NFR-02 ✗, NFR-03 ✗, NFR-04 parcial, R-05 trivial (no hay migración).

**Veredicto**: descartada — no resuelve el problema que la dirección encargó.

---

### Opción 2 — Microservicios descompuestos **por capas técnicas** (web / business / data)

**Descripción**: dividir el sistema en 3 servicios técnicos: API/Web, Business Logic, Data Access. Cada uno escala como pool independiente.

**Pros**

- Escalado horizontal posible por capa.
- Refactor relativamente directo desde un monolito en capas.

**Contras**

- **Anti-patrón** explícito del cap 2 Richardson (*Decompose by technical layer*): un cambio en Order Taking implica modificar las 3 capas y desplegarlas en orden → alta coordinación entre equipos.
- Latencia degrada (cada request salta 3 procesos por la red): rompe NFR-01.
- Las capas no son aislables de fallos (toda capa de negocio cae si un solo flujo la satura): rompe NFR-02.
- Genera un **distributed monolith**: cambios atómicos requieren múltiples deploys coordinados.

**Impacto en NFRs**: NFR-01 ✗, NFR-02 ✗, NFR-03 parcial, NFR-04 ✗.

**Veredicto**: descartada — anti-patrón documentado.

---

### Opción 3 — Microservicios **por business capability** (cap 2 Richardson) — RECOMENDADA

**Descripción**: cada una de las **7 capacidades** del [PRD §3](../prd/PRD.md) (Consumer, Restaurant, Order Taking, Kitchen, Delivery, Billing, Notifications) se convierte en un microservicio con su propia base de datos (DB-per-service, cap 5). La migración usa Strangler Fig: un API gateway redirige progresivamente tráfico desde el monolito legacy hacia los nuevos servicios.

**Pros**

- Cumple la guía explícita del cap 2 (*Decompose by Business Capability*): minimiza acoplamiento porque las capacidades son estables en el tiempo.
- Habilita escalado independiente por capacidad (Y-axis del Scale Cube) → cubre R-01 / NFR-03 directamente.
- Aislamiento de fallos: una caída en Notifications no afecta Order Taking → cubre R-02 / NFR-02.
- Compatible con Strangler Fig (cap 13): se migra una capacidad a la vez, comenzando por Notifications (la más asíncrona y de menor riesgo).
- Stack core puede permanecer Java/Spring Boot por capacidad (NFR-08).

**Contras**

- **7 microservicios desde el día 1** implican mayor coste operativo: 7 pipelines, 7 BDs, observabilidad y on-call distribuidos.
- Riesgo de *distributed monolith* si las capacidades quedan acopladas vía BD compartida o llamadas síncronas anidadas → se mitiga con ADR 0002 (IPC) y DB-per-service.
- Coste de infraestructura inicial mayor; se requiere broker, gateway y service discovery.

**Impacto en NFRs**: NFR-01 ✓ (con cache y BFF), NFR-02 ✓, NFR-03 ✓✓, NFR-04 ✓ (vía broker), NFR-06 requiere Saga (cap 4).

---

### Opción 4 — Microservicios **por subdominio DDD** (Bounded Context puro)

**Descripción**: descomposición por subdominios DDD identificados con Event Storming. Puede dar granularidad distinta a las capabilities (p. ej. dividir Order Taking en "Cart" + "Checkout").

**Pros**

- Maximiza la cohesión de cada servicio (modelo de dominio claro).
- Bounded Contexts explícitos reducen el acoplamiento semántico.
- Compatible con el libro (cap 2 también lo menciona como complemento).

**Contras**

- Requiere **Event Storming + workshops de dominio** antes de migrar → choca con R-05 (migración incremental, tiempo limitado).
- En FTGO los subdominios coinciden ampliamente con las capacidades (no aporta valor diferencial inmediato).
- Mayor riesgo de granularidad excesiva (microservicios "anémicos" que sólo proxyfican CRUD).

**Impacto en NFRs**: NFR-01 ✓, NFR-02 ✓, NFR-03 ✓, NFR-04 ✓, R-05 ✗ (eleva el coste de migración).

**Veredicto**: viable a futuro como **refinamiento** de la opción 3 cuando una capability muestre evidencia clara de tener > 1 bounded context (p. ej. partir Billing en Charging + Payouts).

## 5. Decisión

Se elige la **Opción 3 — Microservicios por business capability** alineada con el cap 2 de Richardson (*Decompose by Business Capability*), aplicando **Strangler Fig** (cap 13) durante 18–24 meses.

**Justificación resumida**

- Es la única opción que cubre simultáneamente R-01 (5x pico), R-02 (99.9 %) y R-03 (latencia) sin caer en anti-patrones del cap 2.
- Las 7 capacidades del [Brief §A.3](../brief/brief.md) son estables a largo plazo, lo que minimiza el riesgo de tener que rediseñar los límites en 12 meses.
- Habilita migración incremental: el orden propuesto es Notifications → Billing → Delivery tracking → Order Taking → Kitchen → Consumer/Restaurant Mgmt, de menor a mayor riesgo de cambio.
- La granularidad por subdominio (Opción 4) queda como refinamiento posterior si la evidencia operativa lo justifica.

## 6. Consecuencias

### 6.1 Positivas

- **Escalabilidad independiente por capacidad** (Y-axis del Scale Cube): cumple NFR-03 directamente.
- **Aislamiento de fallos**: cumple NFR-02 y reduce el blast radius de incidentes en horario pico (BG-02 del BRD).
- **Equipos independientes** capaces de desplegar sin coordinar release trains (BG-04 del BRD).
- **Compatible con Strangler Fig**: el monolito sigue vivo y se reemplaza progresivamente, lo que reduce el riesgo de migración (R-05).
- **Trazabilidad clara** al cap 2 de Richardson y al Brief §A.3.

### 6.2 Negativas

- **Mayor coste operativo inicial**: 7 pipelines, 7 BDs, broker, gateway, tracing distribuido. Requiere capacidad de plataforma (Kubernetes + observabilidad) desde el día 1.
- **Curva de aprendizaje**: el equipo debe absorber patrones nuevos (Saga, outbox, circuit breaker) — riesgo de errores operativos en los primeros 6 meses.
- **Latencia agregada por la red**: una transacción que antes era una llamada local ahora cruza ≥ 2 servicios. Requiere mitigación con BFF / cache (referido a ADR 0002).
- **Consistencia eventual** entre servicios: el FSD UC-04 y UC-06 ya asumen Saga; se acepta NFR-06 con ventana ≤ 2 s p95.
- **Riesgo de distributed monolith** si la próxima ADR sobre IPC elige llamadas síncronas anidadas. Mitigación: ADR 0002 favorecerá patrones asíncronos para integración inter-servicio.

## 7. Follow-ups

- **ADR 0002** (siguiente): mecanismo IPC predominante (síncrono REST vs asíncrono por broker). Esta ADR queda condicionada a que prevalezca el aislamiento de fallos elegido aquí.
- **POC obligatoria**: migrar Notifications fuera del monolito como primer servicio independiente y medir cobertura de observabilidad antes de avanzar a Billing.
- **ADR futuro**: estrategia de datos (DB-per-service, eventos de dominio, CQRS para reporting).
- **ADR futuro**: estrategia de despliegue (Kubernetes, gateway, service mesh — diferido).

## 8. Referencias

- Richardson, C. (2019). *Microservices Patterns*. Manning. Cap 1 (*From Hell* + Scale Cube), Cap 2 (*Decompose by Business Capability* + anti-patrones), Cap 13 (Strangler Fig).
- [PRD](../prd/PRD.md), [FSD](../fsd/FSD.md), [BRD](../brd/BRD.md), [MRD](../mrd/MRD.md).
- [Brief FTGO](../brief/brief.md).
