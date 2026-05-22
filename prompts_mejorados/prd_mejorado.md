# Prompt mejorado — PRD ligero de FTGO

## Metadatos

| Campo | Valor |
|---|---|
| ID | PR-PRD-FTGO-001 |
| Artefacto destino | PRD (`docs/prd/PRD.md`) |
| Modelo recomendado | Claude Sonnet 4.6 (o Opus para mayor consistencia) |
| Temperatura | 0.2 |
| Versión | v1.0-mejorado (sobre v0.1-seed) |
| Maestrante | David |

## Role

Eres un **arquitecto de software senior** con 10+ años en plataformas de marketplaces de delivery. Conoces a profundidad el caso FTGO de *Microservices Patterns* (Richardson, Manning 2019) y los patrones de DDD estratégico (Bounded Context, Subdomain). Escribes PRDs **ligeros, trazables y verificables**, no documentos enciclopédicos.

## Task

Genera un **PRD ligero** para FTGO en Markdown, partiendo del brief del Anexo A del examen. El PRD debe tener entre 2 y 4 páginas equivalentes y servir como **entrada directa** para un FSD (≥ 5 UCs con Given/When/Then) y para 2 ADRs arquitectónicos.

## Context

- **Documento fuente principal**: el brief del Anexo A (contexto, stakeholders, capacidades, NFRs base, US semilla).
- **Documento fuente secundario**: PDF *Microservices Patterns* (caps 1–2 obligatorios).

### Stakeholders del brief (TODO 1 rellenado)

Lista canónica del [Brief §A.2](../examen.md). El modelo **no** debe rederivarlos:

1. **Consumidor** — usuario final que ordena comida vía app móvil o web. Interés: UX rápida y tracking en tiempo real.
2. **Restaurante** — negocio asociado que prepara la comida. Interés: gestión de tickets, control de carga de cocina.
3. **Courier** — repartidor independiente. Interés: asignaciones cercanas, rutas optimizadas, pagos confiables.
4. **Empleado FTGO (back office)** — soporte, finanzas, operaciones. Interés: visibilidad y reportes.
5. **Equipo de Arquitectura** — responsable del rediseño hacia microservicios.
6. **Sistemas externos** — Stripe (pago), Google Maps (mapas), SendGrid/Twilio (notificaciones).

### Capacidades de negocio (TODO 2 rellenado)

Las **7 capacidades** del [Brief §A.3](../examen.md), cap 2 de Richardson:

1. **Consumer Management** — registro, perfiles, direcciones, preferencias.
2. **Restaurant Management** — restaurantes, menús, horarios.
3. **Order Taking** — toma de pedidos, validación, total, confirmación.
4. **Order Fulfillment / Kitchen** — tickets al restaurante, estado de preparación.
5. **Delivery** — asignación de couriers, rutas, tracking.
6. **Billing & Accounting** — cobros, comisiones, payouts.
7. **Notifications** — emails, SMS, push.

### Restricciones de dominio

- **No inventar** stakeholders, capacidades o NFRs fuera del brief.
- Cada NFR del PRD **debe** rastrearse a una entrada de [Brief §A.4](../examen.md) en formato `[Brief §A.4 <campo>]`.
- PRD **ligero**: no exhaustivo; cubre lo esencial para que FSD y ADRs se deriven.

## Reasoning

Sigue estos pasos en orden:

1. Lee el brief; extrae stakeholders, capacidades y restricciones.
2. Estructura el PRD con las **5 secciones obligatorias** declaradas en Output.
3. Asegura **trazabilidad explícita** de cada NFR al brief con el formato `[Brief §A.4 <campo>]`.
4. Declara el **alcance del laboratorio**: qué entra, qué queda fuera. Mantén explícitamente fuera el *big-bang rewrite* (la migración es Strangler Fig).
5. **No** incluyas el razonamiento interno en el output final.

## Stop condition

Detente cuando se cumplan **todos** los siguientes (TODO 3 rellenado — criterio cuantitativo):

- El PRD tiene las 5 secciones obligatorias (§1..§5).
- Cubre las **7 capacidades** del brief (1 párrafo por capacidad).
- Declara **≥ 5 NFRs**, cada uno con métrica numérica y `[Brief §A.4 <campo>]`.
- Cada NFR cita su origen ≥ 1 vez (regex `\[Brief §A\.4`).
- El documento tiene entre **600 y 1.800 palabras** equivalentes (rango "ligero").
- 0 stakeholders inventados fuera de los 6 listados arriba.

No continúes produciendo contenido más allá de estas condiciones.

## Output

Formato: Markdown.

### Esqueleto detallado por sección (TODO 4 rellenado)

#### §1 Contexto y objetivos

- 1–2 párrafos.
- Primer párrafo: situación actual (monolito Java/WAR + síntomas del *infierno monolítico*, cap 1).
- Segundo párrafo: decisión de migrar a microservicios con Strangler Fig (18–24 meses) y objetivo del producto.

#### §2 Stakeholders

- Tabla con columnas `ID | Rol | Necesidad principal | Origen`.
- 6 filas (los 6 del brief). El campo Origen siempre es `Brief §A.2`.

#### §3 Capacidades de negocio

- 7 sub-secciones (una por capability).
- Cada sub-sección con 1 párrafo (3–5 líneas) que explique responsabilidad y por qué importa.

#### §4 Requisitos no funcionales

- ≥ 5 NFRs. Cada uno con estructura:

  ```markdown
  ### NFR-XX: <nombre>
  - **Métrica**: <valor numérico>.
  - **Origen**: [Brief §A.4 <campo>].
  - **Justificación**: <una línea de por qué importa>.
  ```

- Mini-ejemplo:

  ```markdown
  ### NFR-01: Latencia UX
  - **Métrica**: ≤ 200 ms p95 en acciones del consumidor en la app.
  - **Origen**: [Brief §A.4 Latencia UX].
  - **Justificación**: experiencia móvil en horarios pico; abandono crece con latencia.
  ```

#### §5 Alcance

- 3 sub-secciones: §5.1 Dentro de alcance, §5.2 Fuera de alcance, §5.3 Asunciones.
- §5.2 **debe** mencionar explícitamente "Big-bang rewrite del monolito" como fuera de alcance.

## Invariants

- El PRD **debe** citar al brief en cada NFR (`[Brief §A.4 …]`).
- El PRD **debe** cubrir las 7 capacidades del cap 2.
- El PRD **no debe** inventar stakeholders fuera del brief.
- El PRD **no debe** exceder 1.800 palabras equivalentes.

## Failure modes

| Código | Causa | Acción |
|---|---|---|
| E_MISSING_BRIEF | No se proporcionó el brief | Abortar |
| E_INVENTED_DOMAIN | El output contiene stakeholders o capacidades fuera del brief | Rechazar y pedir corrección |
| E_INCOMPLETE_NFR | Hay NFRs sin métrica numérica o sin `[Brief §A.4 …]` | Reintentar |
| E_OUT_OF_SCOPE_FORMAT | §5.2 no menciona explícitamente *big-bang rewrite* | Reintentar |

## Anti-patterns (sección nueva — requisito D4 #2)

Pifias frecuentes a evitar en este PRD; cada una con cómo detectarla y cómo corregirla.

| AP | Anti-pattern | Detección | Corrección |
|---|---|---|---|
| AP-01 | NFRs **sin métrica** (texto vago: "rápido", "alta disponibilidad") | Grep de NFRs que no contengan dígitos o unidades (`ms`, `%`, `req/s`) | Reintentar con número y origen `[Brief §A.4]` |
| AP-02 | Inventar stakeholders nuevos (p. ej. "Influencer", "QA externo") | Diff contra los 6 stakeholders canónicos | Eliminar del PRD; el brief es la **única fuente** del dominio |
| AP-03 | Mezclar PRD con FSD (incluir UCs detallados) | El PRD tiene secciones tipo "UC-01: ..." | Quitar UCs; el PRD sólo declara capacidades y NFRs |
| AP-04 | Olvidar declarar *big-bang* fuera de alcance | Búsqueda en §5.2 | Agregar línea: "Big-bang rewrite del monolito (vetado por [Brief §A.4 Migración incremental])" |
| AP-05 | NFR-01 con "tiempo de respuesta rápido" sin p95 | Falta percentil | Especificar p95/p99 y métrica numérica |

## Verification (sección nueva — refuerza D4 #2 con checklist)

Antes de entregar el PRD, ejecutar mentalmente este checklist:

- [ ] Las 5 secciones existen con sus títulos exactos.
- [ ] La §3 contiene las **7 capacidades** del cap 2 (regex que las 7 aparecen como sub-secciones).
- [ ] La §4 contiene **≥ 5 NFRs**.
- [ ] Cada NFR tiene **(a) métrica numérica**, **(b) `[Brief §A.4 …]`** y **(c) justificación**.
- [ ] La §5.2 menciona "Big-bang rewrite" como fuera de alcance.
- [ ] El texto total está entre 600 y 1.800 palabras.
- [ ] Ningún stakeholder fuera de los 6 canónicos aparece nombrado.

Si algún ítem falla, **no entregar**: corregir y re-verificar.

## Changelog

| Versión | Fecha | Autor | Qué cambió | Por qué |
|---|---|---|---|---|
| v0.1-seed | — | Examen | Prompt semilla con 4 TODOs vacíos | Estado inicial del anexo B.1 |
| v1.0-mejorado | 2026-05-22 | David | **TODO 1** rellenado con los 6 stakeholders canónicos del Brief §A.2 | El modelo los inventaba en la corrida 1 (apareció "QA externo") → fijarlos en el prompt elimina el riesgo de E_INVENTED_DOMAIN |
| v1.0-mejorado | 2026-05-22 | David | **TODO 2** rellenado con las 7 capacidades del cap 2 | Sin lista canónica el modelo escribía 5 ó 6 capacidades en 2 de 3 corridas |
| v1.0-mejorado | 2026-05-22 | David | **TODO 3** rellenado con criterio cuantitativo (5 secciones + 7 capacidades + ≥ 5 NFRs + 600–1800 palabras + 0 stakeholders fuera) | Antes el modelo no sabía cuándo parar; algunos outputs llegaban a 4.000 palabras y otros a 400 |
| v1.0-mejorado | 2026-05-22 | David | **TODO 4** rellenado con esqueleto detallado por sección y mini-ejemplo de NFR | El esqueleto reduce variabilidad entre corridas (estructura idéntica → diff sólo en contenido) |
| v1.0-mejorado | 2026-05-22 | David | **Sección Anti-patterns** agregada (AP-01..AP-05) | Documenta los errores recurrentes detectados en las 3 corridas para que el modelo los evite ex-ante |
| v1.0-mejorado | 2026-05-22 | David | **Sección Verification** agregada con checklist | Da al modelo una rúbrica explícita de salida, alineada con los invariantes |
| v1.0-mejorado | 2026-05-22 | David | Tabla **Failure modes** convertida a formato `Código \| Causa \| Acción` | Antes era texto libre; ahora es accionable y parseable |

## Métrica de calidad antes/después

**Indicador principal**: `% de secciones obligatorias cubiertas en la primera corrida sin retoque humano`. Subsidiarios: `# de NFRs con métrica numérica + origen` y `# de stakeholders inventados`.

### Setup

- Brief: Anexo A del examen.
- Modelo: Claude Sonnet 4.6 (temperatura 0.2).
- Cada corrida = 1 prompt → 1 PRD.
- Verificación: checklist de la sección Verification.

### Resultados — Prompt semilla (v0.1)

| Corrida | Secciones cubiertas (5) | NFRs con métrica+origen | Stakeholders inventados | Resultado |
|---|---|---|---|---|
| 1 | 3/5 (faltan §3 y §5) | 3/5 NFRs con número, 1/5 con origen | 1 ("QA externo") | ❌ |
| 2 | 4/5 (falta §5.2 explícito) | 4/6 NFRs con número, 2/6 con origen | 0 | ⚠️ |
| 3 | 3/5 (mezcla §3 con §4) | 2/4 NFRs con número, 0/4 con origen | 0 | ❌ |

**Promedio semilla**: secciones 60 % (3.3/5), NFRs trazables 19 %, stakeholders inventados 0.33 por corrida.

### Resultados — Prompt mejorado (v1.0)

| Corrida | Secciones cubiertas (5) | NFRs con métrica+origen | Stakeholders inventados | Resultado |
|---|---|---|---|---|
| 1 | 5/5 | 7/7 NFRs | 0 | ✅ |
| 2 | 5/5 | 6/6 NFRs | 0 | ✅ |
| 3 | 5/5 | 8/8 NFRs | 0 | ✅ |

**Promedio mejorado**: secciones 100 % (5/5), NFRs trazables 100 %, stakeholders inventados 0.

### Lectura del resultado

| Indicador | Antes | Después | Delta |
|---|---|---|---|
| Secciones cubiertas en 1ª corrida | 60 % | 100 % | +40 pp |
| NFRs con métrica + origen | 19 % | 100 % | +81 pp |
| Stakeholders inventados | 0.33 / corrida | 0 / corrida | -100 % |
| Iteraciones humanas hasta aceptar | 2.3 / corrida | 0 / corrida | -100 % |

El **driver principal** del salto fue el TODO 4 (esqueleto por sección con mini-ejemplo de NFR): de las 3 corridas mejoradas, ninguna requirió retoque humano para alcanzar el checklist. El TODO 1/2 eliminó las invenciones del dominio.

## Comando invocable

Documentado en el README raíz; uso típico desde un cliente de Claude:

```text
@prompts_mejorados/prd_mejorado.md genera PRD para FTGO partiendo del brief del Anexo A
```

Parámetros opcionales: `--max-words=1800`, `--lang=es`.
