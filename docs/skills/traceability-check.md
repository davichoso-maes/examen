# Skill — `traceability-check`

## Propósito

Verificar la **trazabilidad explícita** entre artefactos (Brief → BRD/MRD → PRD → FSD → ADR → C4). Es la regla más fuerte de la rúbrica: cada decisión arquitectónica debe poder rastrearse a (a) un capítulo del libro Richardson, (b) una restricción del [Brief FTGO](../brief/brief.md) o (c) una user story semilla.

## Inputs

- Todos los artefactos del repo (`docs/**/*.md`, `docs/diagrams/*.mmd`).
- Brief ([`docs/brief/brief.md`](../brief/brief.md)).

## Reglas de trazabilidad

### NFRs del PRD

- Cada NFR **debe** contener exactamente la cadena `[Brief §A.4` seguida del nombre del campo (Carga, Latencia UX, Disponibilidad, etc.).
- Comando útil:

  ```bash
  grep -n "### NFR" docs/prd/PRD.md
  # cada línea NFR debería tener su correspondiente "[Brief §A.4 ...]"
  grep -c "\[Brief §A\.4" docs/prd/PRD.md   # ≥ # de NFRs
  ```

### UCs del FSD

- Cada UC debe declarar campo **Origen** con valor en `{US-01, US-02, US-03, "Richardson cap N", "NFR-XX (PRD)"}`.
- Cada UC debe declarar **Capacidad PRD** (una de las 7 del Brief §A.3).
- Comando útil:

  ```bash
  grep -n "| Capacidad PRD" docs/fsd/FSD.md
  grep -n "| Origen" docs/fsd/FSD.md
  ```

### ADRs

- La sección **Decisión** debe citar ≥ 1 `cap N` del libro de Richardson o ≥ 1 restricción del brief por ID (R-NN).
- Cada opción evaluada debe declarar impacto en `NFR-XX` por ID.
- Comando útil:

  ```bash
  grep -nE "(cap [0-9]+|NFR-[0-9]+|R-[0-9]+)" docs/adr/0001-*.md
  ```

### Diagramas C4

- Los actores y sistemas del C4 Context deben coincidir con los del PRD §2 (Stakeholders) y con las integraciones declaradas en el Brief §A.2.
- Los contenedores del C4 Container deben corresponder a capabilities del PRD §3 (1:1 o agrupados explícitamente).
- El monolito legacy debe aparecer como `System_Ext` (consecuencia de Strangler Fig en ADR 0001).

## Salida esperada

Para cada par (origen, destino) declarar **trazado / no trazado**:

```
Brief §A.2 (Consumidor)       -> PRD §2 SH-01            ✓
Brief §A.4 Latencia UX        -> PRD §4 NFR-01            ✓
US-01 (Toma de pedido)        -> FSD UC-01                ✓
Richardson cap 4 (Saga)       -> FSD UC-04, FSD UC-06     ✓
PRD NFR-04 + cap 3 IPC        -> ADR 0002 Decisión        ✓
ADR 0002 (Kafka)              -> C4 Container (broker)    ✓
```

Si alguna fila reporta ✗, **no hacer commit** y agregar la cita faltante.

## Errores comunes

| Error | Síntoma | Cómo se corrige |
|---|---|---|
| NFR sin `[Brief §A.4 …]` | El reviewer no puede rastrear el origen | Agregar la cita explícita al final del NFR |
| UC derivado sin "Origen" | Penaliza la rúbrica (UCs inventados) | Agregar `Origen: Richardson cap N` o `NFR-XX (PRD)` |
| ADR sin `cap N` ni `R-NN` | Decisión "porque sí" | Citar el patrón aplicado (cap 2, 3, 4, 5, 13) |
| C4 sin monolito legacy | Olvida Strangler Fig | Agregar `System_Ext` del monolito en L1 y línea de coexistencia con Kafka en L2 |

## Referencias

- [Brief FTGO](../brief/brief.md).
- Rúbrica del examen — "Trazabilidad obligatoria".
- Richardson, C. (2019). *Microservices Patterns*. Manning. Caps 1–6, 11, 13.
