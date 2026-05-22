# Skills de soporte

Esta carpeta contiene **skills reproducibles** que documentan procesos repetibles aplicados durante la elaboración de los artefactos de la entrega. No son artefactos evaluables por sí mismos — son la "lente" con la que se construyeron y verificaron los documentos del repo.

## Skills disponibles

| Skill | Propósito | Cuándo usarla |
|---|---|---|
| [`verify-artifact.md`](verify-artifact.md) | Checklist de auto-verificación contra la rúbrica del examen | Antes de cada commit / al cerrar la entrega |
| [`traceability-check.md`](traceability-check.md) | Verifica que cada NFR/UC/decisión cite su origen (Brief / libro / US) | Tras cambios en PRD, FSD o ADRs |

## Convención

- Cada skill es un Markdown autosuficiente con: propósito, inputs, pasos y salida esperada.
- Las skills **no introducen nuevas decisiones**: sólo verifican que las existentes cumplan los invariantes declarados en cada artefacto.
