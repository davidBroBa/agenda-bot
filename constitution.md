# Constitución — Agenda inteligente y recordatorios por Telegram

> Este documento define los principios que **no se negocian** en este proyecto.
> Es un documento vivo: se modifica solo por decisión explícita del responsable,
> y cada cambio queda registrado en el historial.

## 1. Propósito

Sistema personal que gestiona tareas y recordatorios a través de un bot de
Telegram, con persistencia transaccional y avisos que **no se pierden, no se
duplican y se recuperan** tras una caída del servicio.

## 2. Valores

| Valor | Qué significa en la práctica |
|---|---|
| **Simplicidad** | La solución más simple que funciona. Preferir lo que ya existe (n8n, Ollama) sobre lo nuevo y brillante. |
| **Fiabilidad** | Un recordatorio que no llega es un fallo grave. La ruta crítica no depende de componentes frágiles. |
| **Claridad** | El código se lee más que se escribe. El nombre explica la intención. |
| **Seguridad** | Seguro por omisión. El bot es privado: solo usuarios autorizados, nada expuesto a Internet. |
| **Privacidad** | Solo datos de agenda del propio usuario. El modelo de lenguaje es local; no salen datos a terceros. |
| **Honestidad** | Si algo falla, se reporta el fallo real con su comando. Nunca se maquilla un resultado. |

## 3. Principios de diseño

### 3.1 Arquitectura
- Una responsabilidad por módulo. El acoplamiento se mantiene bajo control.
- El servicio `core` es la **única** capa que toca el esquema de datos `agenda`.
- La ruta crítica de los recordatorios (el worker) **no depende de n8n**.
- Las decisiones con alternativas reales van en `docs/adr/`.
- Todo se levanta de forma reproducible con Docker Compose.

### 3.2 Código
- Tipado estricto: `mypy --strict`. Prohibido `Any`, `# type: ignore` y
  `cast` sin justificación escrita.
- Funciones cortas, una responsabilidad.
- Nombres en inglés, comentarios y documentación en español.
- Nada de lógica de negocio en controladores (routers FastAPI) ni en los
  workflows de n8n: la lógica de negocio vive en `agenda.domain`.
- Toda función pública lleva docstring que diga **qué** hace, no cómo.

### 3.3 Testing
- **Test-first**: el test se escribe antes que la implementación y se ve fallar.
- Cobertura mínima: **80%** en la lógica de negocio y de fechas/recurrencia.
- Toda lógica de negocio tiene test: camino feliz, casos límite, casos de error.
- **Tests en rojo bloquean.** No se cierra una tarea con tests en rojo.
- Los casos de zona horaria, DST y recurrencia son **obligatorios**.

### 3.4 SDD: la spec manda
- Enfoque: **spec-anchored**. La spec se mantiene viva y gobierna el código.
- Una funcionalidad = una carpeta `specs/NNN-nombre/` con `spec.md`, `plan.md` y `tasks.md`.
- Los requisitos se escriben en **EARS**: `CUANDO ... EL SISTEMA ...`,
  `SI ... ENTONCES EL SISTEMA ...`, `MIENTRAS ... EL SISTEMA ...`,
  `EL SISTEMA ...`. Un requisito sin criterio verificable no es un requisito.
- Si algo no está en la spec, no se implementa. Si falta una decisión, se para
  y se pregunta.
- Un cambio de requisito se escribe **primero en la spec**, luego en el plan,
  luego en las tareas, y por último en el código.

### 3.5 Documentación
- El SDD se actualiza cuando cambia la arquitectura, no "eventualmente".
- Cada criterio de aceptación es verificable: incluye su comando o su test.
- Los diagramas van en ASCII plano, nunca con caracteres de dibujo.
- Existe una bitácora (`docs/BITACORA.md`) que registra, por fecha, qué se hizo,
  con qué comando se verificó y qué queda pendiente.

## 4. Reglas de codificación

### Nombres
- Sustantivos para datos, verbos para acciones, `get`/`create`/`update`/`delete`
  para operaciones.
- Sin abreviaturas crudas (`usr`, `cfg`, `mgr`). Palabras completas.

### Formato
- `ruff format` + `ruff check` (warnings como errores). El formateo no se debate
  en revisión de código.

### Estructura
```
core/                 # paquete Python `agenda`: API, worker y gateway
  src/agenda/
    api/              # routers FastAPI, sin lógica de negocio
    domain/           # lógica de negocio (tareas, recurrencia, políticas)
    worker/           # motor de recordatorios
    gateway/          # long polling de Telegram
    db/               # acceso a datos y migraciones
    nlp/              # parsing de lenguaje natural (Ollama)
    config/           # configuración tipada desde entorno
  tests/
db/migrations/        # migraciones SQL versionadas
n8n/workflows/        # workflows versionados como JSON
ops/                  # backup, restore, healthcheck
docs/
```

## 5. Procesos

### Decisiones
- Toda decisión difícil de revertir requiere un ADR **antes** de implementarla.
- El ADR incluye alternativas y consecuencias negativas. Sin negativas, no está pensado.

### Code review
- Checklist obligatorio: seguridad, tipos, tests, manejo de fechas, rendimiento.
- Los gates críticos bloquean el merge.

### Releases
- Versionado semántico.
- `CHANGELOG.md` actualizado en cada release.

## 6. Estándares de calidad

| Área | Estándar |
|---|---|
| Accesibilidad | No aplica: no hay interfaz propia. La interfaz es Telegram y se hereda su accesibilidad; los mensajes se escriben claros y cortos. |
| Seguridad | OWASP Top 10, sin vulnerabilidades altas (`pip-audit`). |
| Rendimiento | Un `/tick` del worker por debajo de 200 ms en el caso normal. |
| Tipado | `mypy --strict`, cero errores, cero `Any`. |
| Cobertura | ≥ 80% en lógica de negocio. |
| Privacidad | Cero datos hacia terceros. El modelo es local (Ollama). |
| Requisitos | 100% de los RF en EARS y verificables. |
| Verificación | Todo gate crítico en verde antes de entregar. |

## 7. Responsabilidades

| Rol | Responsabilidad |
|---|---|
| Responsable del proyecto (David) | Aprueba SDD, modelo de amenazas, decisiones y releases |
| Desarrollo (agente + subagentes) | Implementa, testea y documenta |
| Revisión (`sdd-reviewer`, auditores) | Valida gates, seguridad y cumplimiento de la spec |

## 8. Evolución

Esta constitución se modifica cuando el proyecto aprende algo. Cada cambio:

1. Se propone con el motivo concreto.
2. Se aprueba explícitamente.
3. Se registra en el historial con fecha y responsable.

## Historial

| Fecha | Versión | Cambio | Responsable |
|---|---|---|---|
| 2026-10-09 | 1.0 | Versión inicial | David |
