# Examen Práctico — Caso FTGO (Food To Go)

**Maestrante**: Antonio David Ovando Arcienega

---

## 1. Propósito

Este repositorio documenta la **arquitectura objetivo** del marketplace de delivery FTGO en su migración del monolito Java/WAR hacia microservicios (Strangler Fig, 18–24 meses). Contiene:

- **BRD / MRD / PRD / FSD**: trazabilidad de negocio → mercado → producto → función.
- **ADRs**: decisiones arquitectónicas con opciones, trade-offs y consecuencias.
- **Diagramas C4**: Context (nivel 1) y Container (nivel 2) en Mermaid.
- **Prompts mejorados**: 2 de los 4 prompts semilla (PRD y ADR) con cambios documentados, métricas y comandos invocables. Las versiones semilla originales se conservan en `prompts_semilla/`.

## 2. Estructura del repositorio

```
.
├── README.md                          # Este archivo
├── docs/
│   ├── brief/brief.md                 # Brief FTGO (única fuente del dominio)
│   ├── brd/BRD.md                     # Business Requirements Document
│   ├── mrd/MRD.md                     # Market Requirements Document
│   ├── prd/PRD.md                     # Product Requirements Document
│   ├── fsd/FSD.md                     # Functional Specification (≥ 5 UCs con GWT)
│   ├── adr/
│   │   ├── 0001-estilo-arquitectonico.md
│   │   └── 0002-mecanismo-ipc.md
│   ├── diagrams/
│   │   ├── c4_context.mmd             # C4 nivel 1
│   │   └── c4_container.mmd           # C4 nivel 2
│   └── skills/                        # Skills de verificación (verify-artifact, traceability-check)
├── prompts_semilla/                   # Prompts semilla originales (v0.1-seed, con TODOs)
│   ├── prd_semilla.md
│   ├── fsd_semilla.md
│   ├── adr_semilla.md
│   └── c4_semilla.md
└── prompts_mejorados/
    ├── prd_mejorado.md                # Mejora del prompt PRD semilla
    └── adr_mejorado.md                # Mejora del prompt ADR semilla
```

## 3. Mapa de trazabilidad

```
Brief FTGO (docs/brief/brief.md)
        │
        ▼
   BRD → MRD → PRD ──► NFRs
                  │
                  ├──► FSD (≥ 5 UCs con Given/When/Then)
                  │
                  └──► ADR 0001 (estilo) ──► ADR 0002 (IPC)
                                                │
                                                ▼
                                       C4 Context (L1)
                                       C4 Container (L2)
```

Cada NFR del PRD cita `[Brief §A.4]`. Cada UC del FSD cita su origen (US-NN o capacidad PRD o capítulo Richardson). Cada decisión ADR cita restricciones del brief y capítulos del libro.

## 4. Comandos invocables (prompts mejorados)

Los prompts pueden ejecutarse referenciando el archivo desde un cliente compatible con Claude / Cursor / Continue:

```text
# Generar el PRD desde el brief
@prompts_mejorados/prd_mejorado.md genera PRD para FTGO

# Generar un ADR (parámetro = decisión a tomar)
@prompts_mejorados/adr_mejorado.md decision="estilo arquitectónico"
```

Detalle completo y métricas antes/después con 3 corridas en cada archivo de `prompts_mejorados/`.

## 5. Métricas declaradas (resumen)

| Prompt | Indicador medido | Antes (semilla) | Después (mejorado) |
|---|---|---|---|
| `prd_mejorado.md` | % secciones cubiertas en primera corrida | 60 % (3/5) | 100 % (5/5) |
| `adr_mejorado.md` | # opciones reales evaluadas + ambas consecuencias | 1.7 promedio / no | 3.0 promedio / sí |

Evidencia por corrida en cada archivo de `prompts_mejorados/<nombre>.md` § Métrica.

## 6. Cómo reproducir

1. Leer el [Brief FTGO](docs/brief/brief.md) (única fuente del dominio) y los prompts semilla en `prompts_semilla/`.
2. Leer artefactos en orden: BRD → MRD → PRD → FSD → ADRs → diagramas C4.
3. Para regenerar un artefacto: invocar el prompt correspondiente de `prompts_mejorados/` con los inputs declarados en su sección **Context**.
4. Render de diagramas: cualquier visor Mermaid con soporte C4 (Mermaid ≥ 10.0).
5. Antes de cada commit aplicar las skills de verificación:
   - [`docs/skills/verify-artifact.md`](docs/skills/verify-artifact.md) — checklist por tipo de artefacto.
   - [`docs/skills/traceability-check.md`](docs/skills/traceability-check.md) — verificación de trazabilidad explícita Brief → PRD → FSD → ADRs → C4.

## 6.1 Atajos de navegación

- [Brief FTGO](docs/brief/brief.md)
- [BRD](docs/brd/BRD.md) · [MRD](docs/mrd/MRD.md) · [PRD](docs/prd/PRD.md) · [FSD](docs/fsd/FSD.md)
- [ADR 0001 — Estilo arquitectónico](docs/adr/0001-estilo-arquitectonico.md)
- [ADR 0002 — Mecanismo IPC](docs/adr/0002-mecanismo-ipc.md)
- [C4 Context](docs/diagrams/c4_context.mmd) · [C4 Container](docs/diagrams/c4_container.mmd)
- [Prompt mejorado PRD](prompts_mejorados/prd_mejorado.md) · [Prompt mejorado ADR](prompts_mejorados/adr_mejorado.md)

## 7. Self-check de entrega

- [x] Branch `release/exam-lab` creada.
- [x] Estructura de carpetas conforme al árbol pedido.
- [x] PRD con 5 secciones obligatorias y NFRs trazables.
- [x] FSD con ≥ 5 UCs y bloques Given/When/Then.
- [x] 2 ADRs con ≥ 3 opciones, trade-offs y consecuencias positivas y negativas.
- [x] 2 diagramas C4 válidos en Mermaid con tecnología/protocolo en cada relación.
- [x] 2 prompts mejorados con changelog, métrica de 3 corridas y comando invocable.
- [x] README ejecutable.
