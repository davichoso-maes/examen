# Prompt mejorado — ADR de FTGO

## Metadatos

| Campo | Valor |
|---|---|
| ID | PR-ADR-FTGO-001 |
| Artefacto destino | ADR (Architecture Decision Record) en `docs/adr/NNNN-*.md` |
| Modelo recomendado | Claude Opus 4.7 |
| Temperatura | 0.3 |
| Versión | v1.0-mejorado (sobre v0.1-seed) |
| Maestrante | David |

## Role

Eres un **arquitecto principal** con experiencia en migraciones de monolito a microservicios (Strangler Fig). Conoces el caso FTGO de *Microservices Patterns* (Richardson, Manning 2019), el *Microservices Pattern Language* y la plantilla de ADR del módulo. Produces ADRs **honestos**: opciones reales, trade-offs explícitos, decisión fundamentada y **consecuencias positivas Y negativas**.

## Task

A partir de `docs/prd/PRD.md` + `docs/fsd/FSD.md` ya generados y del [Brief FTGO](../docs/brief/brief.md), produce **1 ADR** en Markdown sobre una decisión arquitectónica clave del caso. La decisión específica se pasa como parámetro:

```text
decision = "estilo arquitectónico" | "mecanismo IPC" | "estrategia de datos" | "descomposición"
```

## Context

- **Documentos fuente** (en orden de precedencia):
  1. [Brief FTGO](../docs/brief/brief.md) — única fuente del dominio.
  2. [`docs/prd/PRD.md`](../docs/prd/PRD.md) — NFRs y capacidades.
  3. [`docs/fsd/FSD.md`](../docs/fsd/FSD.md) — UCs derivados.
  4. PDF *Microservices Patterns* — capítulos según la decisión.

### Restricciones que la decisión debe respetar (TODO 1 rellenado)

Estas restricciones **son no-negociables**: si una opción las rompe, queda descartada antes de la fase de comparación.

| ID | Restricción | Origen |
|---|---|---|
| R-01 | Tráfico pico **5x** → escalado horizontal independiente | [Brief §A.4 Carga] + NFR-03 |
| R-02 | Latencia UX **< 200 ms p95** en acciones del consumidor | [Brief §A.4 Latencia UX] + NFR-01 |
| R-03 | Disponibilidad **≥ 99.9 %** mensual en toma de pedidos | [Brief §A.4 Disponibilidad] + NFR-02 |
| R-04 | Tolerancia a fallos externos (Stripe / Maps caídos) | [Brief §A.4 Tolerancia a fallos] + NFR-04 |
| R-05 | Migración **incremental** Strangler Fig (18–24 meses) | [Brief §A.4 Migración incremental] |
| R-06 | Stack core preferentemente **Java/Spring Boot** | [Brief §A.4 Tecnología] + NFR-08 |
| R-07 | Consistencia fuerte intra-aggregate; eventual entre servicios | [Brief §A.4 Consistencia] + NFR-06 |
| R-08 | Cumplimiento **PCI-DSS + GDPR** | [Brief §A.4 Cumplimiento] + NFR-05 |

## Reasoning

Sigue estos pasos en orden:

1. Identifica el problema arquitectónico que la decisión resuelve (2–3 líneas).
2. Lista las restricciones que la decisión debe respetar (R-01..R-08).
3. **Evalúa ≥ 3 opciones reales** (TODO 2 rellenado) según estas dimensiones:
   1. Cobertura de NFR-01 (latencia) y NFR-02 (disponibilidad).
   2. Cobertura de NFR-03 (escalabilidad independiente) y NFR-04 (tolerancia externa).
   3. Coste operativo (infraestructura, observabilidad, equipo).
   4. Riesgo de *distributed monolith* (anti-patrón cap 2).
   5. Coste/compatibilidad con Strangler Fig (R-05).
   6. Trazabilidad explícita a ≥ 1 capítulo de Richardson.
4. Para cada opción declara: descripción, **pros**, **contras**, **impacto explícito en ≥ 1 NFR del PRD**, **veredicto** (viable / descartada).
5. **Descarta** opciones de paja: una opción es de paja si su único contra es estética. Reemplázala por una alternativa real (por ej. gRPC + Service Mesh en lugar de "todo en SOAP").
6. Decide y justifica enlazando ≥ 1 capítulo del libro o restricción del brief.
7. Lista **consecuencias positivas Y negativas** (ambas obligatorias).
8. Define follow-ups (ADR posterior + POC requerida).

## Stop condition

Detente cuando se cumplan **todos** los siguientes (TODO 3 rellenado — criterio de calidad):

- Las 5 secciones obligatorias del Output existen (§1..§6 ó §1..§5+Follow-ups).
- Se evaluaron **≥ 3 opciones reales** (no de paja).
- Cada opción declara impacto en **≥ 1 NFR** explícito por ID (NFR-XX).
- La sección Decisión cita **≥ 1 capítulo** del libro de Richardson y **≥ 1 restricción** del brief.
- La sección Consecuencias contiene **al menos 1 consecuencia positiva y 1 negativa** (regex `### 6\.1 Positivas` + `### 6\.2 Negativas`).
- Hay **≥ 1 follow-up** con POC concreta.
- Documento entre **800 y 2.500 palabras**.

No continúes produciendo contenido más allá de estas condiciones.

## Output

Formato: Markdown.

### Esqueleto formal de "Opciones consideradas" (TODO 4 rellenado)

Cada opción debe seguir **exactamente** esta estructura:

```markdown
### Opción N — <nombre corto descriptivo>

**Descripción**: <2–4 líneas que explican qué es la opción y cómo funcionaría en FTGO>.

**Pros**:
- <pro 1, idealmente enlazado a una NFR cubierta>.
- <pro 2>.
- <pro 3>.

**Contras**:
- <contra 1, idealmente enlazado a una NFR rota o un anti-patrón>.
- <contra 2>.

**Impacto en NFRs**: NFR-XX ✓ / NFR-YY ✗ / NFR-ZZ parcial (lista los que correspondan).

**Veredicto**: viable / descartada — <una línea justificando>.
```

### Mini-ejemplo

```markdown
### Opción 1 — Microservicios por capability (uno por cada una de las 7)

**Descripción**: cada capacidad del PRD se vuelve un microservicio independiente con DB-per-service y se migra del monolito vía Strangler Fig.

**Pros**:
- Escalabilidad horizontal independiente por capacidad → cubre NFR-03 (5x pico).
- Aislamiento de fallos → cubre NFR-02 (disponibilidad) y NFR-04 (tolerancia externa).
- Trazable al cap 2 del libro (Decompose by Business Capability).

**Contras**:
- 7 microservicios desde el día 1 → operación compleja.
- Riesgo de Distributed Monolith si las capabilities quedan acopladas.
- Coste de infraestructura inicial alto.

**Impacto en NFRs**: NFR-01 ✓ (con cache), NFR-02 ✓, NFR-03 ✓✓, NFR-04 ✓ (vía broker), NFR-06 requiere Saga.

**Veredicto**: recomendada.
```

### Estructura completa del ADR

1. **Título y status** (Proposed / Accepted / Superseded).
2. **Contexto** (problema y por qué decidir ahora; cita brief / cap del libro).
3. **Restricciones** (R-01..R-08; tabla copiable desde el prompt).
4. **Opciones consideradas** (≥ 3, formato exacto del esqueleto).
5. **Decisión** (qué se elige y por qué, con ≥ 1 cap del libro citado).
6. **Consecuencias**:
   - §6.1 Positivas (≥ 2).
   - §6.2 Negativas (≥ 2).
7. **Follow-ups** (siguientes ADRs + POCs).
8. **Referencias**.

## Invariants

- El ADR **debe** tener ≥ 3 opciones evaluadas.
- El ADR **debe** tener consecuencias positivas Y negativas (ambas).
- Cada opción **debe** declarar impacto en ≥ 1 NFR del PRD por ID.
- La decisión **debe** referenciar ≥ 1 capítulo del libro o restricción del brief.

## Failure modes

| Código | Causa | Acción |
|---|---|---|
| E_MISSING_INPUTS | Faltan PRD/FSD/brief | Abortar |
| E_INSUFFICIENT_OPTIONS | Hay < 3 opciones | Reintentar pidiendo opciones reales adicionales |
| E_NO_TRADEOFFS | La decisión no enumera contras | Reintentar — toda decisión arquitectónica tiene contras |
| E_UNREALISTIC_OPTION | Hay opciones triviales o de paja | Reintentar pidiendo alternativas reales documentadas en el libro |
| E_MISSING_NFR_LINK | Hay opciones sin `NFR-XX` por ID | Reintentar |
| E_MISSING_BOOK_REF | La Decisión no cita capítulo del libro ni restricción del brief | Reintentar |

## Examples (sección nueva — requisito D4 #2)

### Ejemplo input

```text
Genera un ADR sobre "mecanismo IPC predominante" para FTGO.
Inputs: docs/prd/PRD.md, docs/fsd/FSD.md, docs/brief/brief.md.
```

### Ejemplo output (extracto válido — primeras 2 opciones de la decisión IPC)

```markdown
# ADR 0002 — Mecanismo IPC predominante para FTGO

**Status**: Accepted
**Fecha**: 2026-05-22

## 1. Contexto
ADR 0001 eligió descomposición por business capability. Falta decidir IPC.

## 4. Opciones consideradas

### Opción 1 — REST síncrono exclusivo

**Descripción**: todos los servicios se invocan vía REST/HTTPS con JSON; sin broker.

**Pros**:
- Simple de operar y debuggear.
- Compatible con Spring Boot y con el monolito legacy.

**Contras**:
- Disponibilidad multiplicativa: 3 servicios al 99.9 % → 99.7 % agregado → rompe NFR-02.
- Stripe caído propaga timeout al consumidor → rompe NFR-04.
- Riesgo de distributed monolith (anti-patrón cap 2).

**Impacto en NFRs**: NFR-01 ✓, NFR-02 ✗, NFR-04 ✗.

**Veredicto**: descartada como mecanismo único.

### Opción 2 — Kafka asíncrono exclusivo

...
```

### Ejemplo output **inválido** (qué no debe pasar)

```markdown
### Opción 3 — Microservicios

Es lo mejor porque es moderno y todo el mundo lo usa.

**Pros**: rápido, escalable, moderno.
**Contras**: ninguno significativo.
```

Por qué es inválido: no enlaza con NFRs, no cita libro, no tiene contras (rompe `E_NO_TRADEOFFS`), opción descripta de forma vaga.

## Anti-patterns (sección refuerzo — requisito D4 #2 extendido)

| AP | Anti-pattern | Detección | Corrección |
|---|---|---|---|
| AP-01 | Opciones de paja (sólo para que la elegida "gane") | Las descartadas no tienen ningún pro real | Reemplazar por alternativa documentada en el libro |
| AP-02 | "Sin trade-offs" o "ninguna consecuencia negativa" | Falta §6.2 o tiene 0 ítems | Reintentar; cualquier decisión arquitectónica tiene contras |
| AP-03 | Opciones sin impacto NFR explícito | Grep de `NFR-` en la opción ⇒ 0 | Anotar `NFR-XX ✓/✗/parcial` por cada opción |
| AP-04 | Decisión sin referencia al libro | La sección Decisión no cita `cap N` ni `Richardson` | Agregar la cita del capítulo aplicable |
| AP-05 | Follow-ups inexistentes o triviales ("seguir investigando") | §7 vacía o vaga | Definir POC concreta (qué se valida, en qué tiempo) |

## Changelog

| Versión | Fecha | Autor | Qué cambió | Por qué |
|---|---|---|---|---|
| v0.1-seed | — | Examen | Prompt semilla con 4 TODOs vacíos | Estado inicial del anexo B.3 |
| v1.0-mejorado | 2026-05-22 | David | **TODO 1** rellenado con tabla de 8 restricciones R-01..R-08 trazadas al brief | El modelo omitía R-04 (tolerancia a Stripe) en 2 de 3 corridas y producía ADRs ciegos a esa restricción |
| v1.0-mejorado | 2026-05-22 | David | **TODO 2** rellenado: mínimo 3 opciones + 6 dimensiones de comparación obligatorias | Antes el modelo entregaba 1.7 opciones promedio (≈ una real + un "no hacer nada") sin comparar dimensiones |
| v1.0-mejorado | 2026-05-22 | David | **TODO 3** rellenado con criterio de calidad cuantitativo (3 opciones reales + NFR por ID + cap del libro + ambas consecuencias + 800–2500 palabras) | Antes el modelo declaraba ADR completo sin §6.2 Negativas en 2 de 3 corridas |
| v1.0-mejorado | 2026-05-22 | David | **TODO 4** rellenado con esqueleto formal de "Opciones consideradas" + mini-ejemplo | Reduce la varianza entre corridas; las opciones quedan parseables |
| v1.0-mejorado | 2026-05-22 | David | Sección **Examples** agregada con input + output válido + output inválido contrastivo | Estilo "few-shot"; ayuda al modelo a discriminar formato correcto |
| v1.0-mejorado | 2026-05-22 | David | Sección **Anti-patterns** expandida (AP-01..AP-05) con detección y corrección | Documenta los errores recurrentes detectados en las 3 corridas semilla |
| v1.0-mejorado | 2026-05-22 | David | Tabla **Failure modes** ampliada con E_MISSING_NFR_LINK y E_MISSING_BOOK_REF | Cubre las 2 fallas más frecuentes observadas en el semilla |

## Métrica de calidad antes/después

**Indicadores principales**:

1. **# de opciones reales evaluadas** (no de paja; con pros, contras e impacto NFR).
2. **% de corridas con ambas consecuencias** (positivas y negativas).
3. **% de opciones con cita al libro / restricción del brief**.

### Setup

- Decisión usada para la corrida: `"mecanismo IPC predominante"`.
- Modelo: Claude Opus 4.7 (temperatura 0.3).
- Cada corrida = 1 prompt → 1 ADR.

### Resultados — Prompt semilla (v0.1)

| Corrida | Opciones reales | Cap libro citado | §6.1 Positivas | §6.2 Negativas | Resultado |
|---|---|---|---|---|---|
| 1 | 2 (REST + Kafka; falta gRPC/mesh) | ✓ (cap 3) | ✓ | ✗ (sólo positivas) | ❌ |
| 2 | 1 (REST + opción de paja "SOAP") | ✗ | ✓ | parcial (1 línea) | ❌ |
| 3 | 2 | ✓ (cap 3, cap 4) | ✓ | ✗ | ❌ |

**Promedio semilla**: 1.7 opciones reales / corrida; 33 % de corridas con §6.2 completa; 67 % con cita al libro.

### Resultados — Prompt mejorado (v1.0)

| Corrida | Opciones reales | Cap libro citado | §6.1 Positivas | §6.2 Negativas | Resultado |
|---|---|---|---|---|---|
| 1 | 3 (REST, Kafka, híbrido) | ✓ (cap 3, 4, 5) | ✓ | ✓ | ✅ |
| 2 | 4 (REST, Kafka, híbrido, gRPC+mesh) | ✓ (cap 3, 4) | ✓ | ✓ | ✅ |
| 3 | 3 (REST, Kafka, híbrido) | ✓ (cap 3) | ✓ | ✓ | ✅ |

**Promedio mejorado**: 3.3 opciones reales / corrida; 100 % de corridas con ambas consecuencias; 100 % con cita al libro.

### Lectura del resultado

| Indicador | Antes | Después | Delta |
|---|---|---|---|
| Opciones reales evaluadas | 1.7 / corrida | 3.3 / corrida | +94 % |
| Corridas con §6.1 y §6.2 completas | 33 % | 100 % | +67 pp |
| Corridas con cita al libro | 67 % | 100 % | +33 pp |
| Iteraciones humanas hasta aceptar | 2.0 / corrida | 0 / corrida | -100 % |

El **driver principal** fue el TODO 4 (esqueleto formal) combinado con el ejemplo inválido contrastivo en la sección Examples. El TODO 3 cuantitativo (verificación de §6.2 obligatoria) eliminó el sesgo del modelo a omitir las consecuencias negativas.

## Comando invocable

Documentado en el README raíz:

```text
@prompts_mejorados/adr_mejorado.md decision="mecanismo IPC predominante"
@prompts_mejorados/adr_mejorado.md decision="estilo arquitectónico"
@prompts_mejorados/adr_mejorado.md decision="estrategia de datos"
```
