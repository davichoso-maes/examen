# Skill — `verify-artifact`

## Propósito

Verificar que un artefacto del repo (PRD, FSD, ADR, C4) cumple la rúbrica del examen **antes** de hacer commit. Pensada para invocarse manualmente o desde un hook pre-commit.

## Inputs

- Ruta del artefacto (`docs/prd/PRD.md`, `docs/fsd/FSD.md`, `docs/adr/NNNN-*.md`, `docs/diagrams/c4_*.mmd`).
- Brief (`examen.md`) como referencia de origen.

## Checklist por tipo de artefacto

### PRD (`docs/prd/PRD.md`)

- [ ] 5 secciones presentes: §1 Contexto, §2 Stakeholders, §3 Capacidades, §4 NFRs, §5 Alcance.
- [ ] §3 cubre las **7 capacidades** del Brief §A.3 (1 párrafo cada una).
- [ ] §4 declara **≥ 5 NFRs**.
- [ ] Cada NFR tiene métrica numérica + `[Brief §A.4 …]` + justificación.
- [ ] §5.2 menciona "big-bang rewrite" como fuera de alcance.
- [ ] 0 stakeholders inventados fuera de los 6 canónicos.

### FSD (`docs/fsd/FSD.md`)

- [ ] **≥ 5 UCs** detallados.
- [ ] Cada UC tiene **al menos 1 bloque Given/When/Then** explícito.
- [ ] Cada UC mapea a una capacidad del PRD.
- [ ] Los UCs derivados (no US semilla) citan origen (libro o NFR).
- [ ] Existe una regla de granularidad documentada.

### ADR (`docs/adr/NNNN-*.md`)

- [ ] Status declarado (Proposed / Accepted / Superseded).
- [ ] **≥ 3 opciones** evaluadas con pros, contras e impacto NFR explícito (`NFR-XX`).
- [ ] Decisión cita ≥ 1 capítulo del libro de Richardson.
- [ ] §Consecuencias contiene §6.1 Positivas **y** §6.2 Negativas (≥ 2 cada una).
- [ ] Al menos 1 follow-up con POC concreta.

### Diagramas C4 (`docs/diagrams/c4_*.mmd`)

- [ ] Sintaxis Mermaid C4 válida (renderiza en Mermaid ≥ 10.0).
- [ ] **Context (L1)**: ≥ 1 Person, ≥ 2 System_Ext, 1 System.
- [ ] **Container (L2)**: ≥ 5 contenedores dentro del System_Boundary.
- [ ] Toda relación de L2 declara **tecnología + protocolo** (`"…", "JSON/HTTPS"`).
- [ ] Coherencia con los ADRs (si elegiste async, el broker aparece; si elegiste DB-per-service, hay N BDs).

## Salida esperada

Ejecutar el checklist mentalmente o vía `grep` y reportar:

```
PRD: 6/6 OK
FSD: 5/5 OK (UC-01..UC-06 con GWT)
ADR 0001: 5/5 OK
ADR 0002: 5/5 OK
C4 Context: 5/5 OK
C4 Container: 5/5 OK
==> Listo para commit
```

Si algún ítem falla, **no hacer commit**: corregir el artefacto y re-verificar.

## Referencias

- Rúbrica del examen (`examen.md`, sección "Rúbrica").
- Invariants declarados en cada artefacto.
