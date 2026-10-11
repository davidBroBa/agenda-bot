# AGENTS.md — agenda-bot

> Se carga al trabajar en este repositorio. Las reglas generales viven en el
> `AGENTS.md` global (`~/.config/opencode`); aquí va solo lo que **difiere**.
> El detalle largo está en `docs/`.

## Qué es y para qué

Asistente personal de agenda y recordatorios por Telegram. Recibe tareas por
comandos o en lenguaje natural y avisa en el momento correcto. Un solo usuario
(David). **No hay interfaz web**: la interfaz es el chat de Telegram.

## Stack y estructura

- Recepción: `agenda-gateway` (long polling de Telegram, Python 3.12).
- Lógica y API: `agenda-core` (FastAPI `:8080`, solo red interna).
- Recordatorios: `agenda-worker` (proceso propio, **no** n8n).
- Flujos: n8n **existente** (se reutiliza, no se recrea).
- IA: Ollama local `qwen3:8b` para el parseo de lenguaje natural.
- Datos: PostgreSQL 17 en contenedor, red interna, sin puerto publicado.

```
app/       núcleo: API, dominio, repositorios, migraciones
worker/    motor de recordatorios
gateway/   long polling de Telegram
n8n/       workflows exportados como JSON (fuente de verdad)
specs/     spec.md, plan.md y tasks.md por funcionalidad
docs/      SDD, THREATS, NETWORK, AI, HERRAMIENTAS, adr/
```

## Comandos

| Acción | Comando |
|---|---|
| Levantar | `docker compose up -d` |
| Tests | `pytest -q` |
| Gates | `powershell -ExecutionPolicy Bypass -File <global>/skills/proyecto-estandar/scripts/check-gates.ps1` |
| Hooks | `powershell -ExecutionPolicy Bypass -File scripts/install-hooks.ps1 -VendorGates` |

> Lint, typecheck y build se fijan al crear el andamiaje en la Fase 1.

## Convenciones

- Código, identificadores, commits y mensajes de log en inglés; textos del bot y
  documentación en español.
- Fechas siempre `timestamptz` en UTC + zona IANA del usuario (`zoneinfo`); nunca
  hora local suelta.
- SQL parametrizado con `psycopg`; cero concatenación de strings.
- Errores tipados y explícitos; prohibido `except: pass` y `print` en producción.
- Toda entrada externa (Telegram, n8n, Ollama) se valida con schema en el borde.

## Reglas de dominio y trampas conocidas

- **No recrear el contenedor n8n existente**: perdería workflows y credenciales
  (sus datos viven en la capa del contenedor). Tocar n8n solo con autorización.
- Idempotencia real: `reminders.dedupe_key` y `reminder_sends.idempotency_key`
  son `UNIQUE`; un reintento no debe duplicar avisos ni tareas.
- Completar o cancelar una tarea **detiene** sus avisos futuros, no solo cambia estado.
- Solo los IDs de Telegram en `users` están autorizados; cualquier otro se ignora
  en silencio.
- Tras una caída, el worker reclama avisos atrasados sin duplicarlos.
- Zona horaria y DST: usar `zoneinfo`; jamás calcular offsets a mano.

## Documentación

| Documento | Contenido |
|---|---|
| `constitution.md` | Principios innegociables |
| `docs/SDD.md` | Arquitectura del proyecto |
| `docs/THREATS.md` | Modelo de amenazas |
| `docs/NETWORK.md` | Topología sin tráfico entrante |
| `docs/AI.md` | Capa de IA (Ollama) y guardrails |
| `docs/HERRAMIENTAS.md` | Skills y MCP recomendados |
| `docs/adr/` | Decisiones arquitectónicas |
| `specs/NNN-nombre/` | spec, plan y tasks por funcionalidad |
| `MEMORY.md` | Memoria entre sesiones |

## Forma de trabajar

- Lee `constitution.md` y la spec activa **antes** de tocar código.
- Nada de código que no esté en la spec activa; si falta una decisión, para y pregunta.
- **Se trabaja en el servidor** (`srv:~/proyectos/agenda-bot`), no en local. Se
  gestiona con `srv.ps1` (`exec`, `put`, `get`). **Nunca `sudo`.**
- Al terminar una fase, actualiza `MEMORY.md` y los checkboxes de `tasks.md`.

### Enforcement

| Capa | Estado | Qué implica para ti |
|---|---|---|
| Hook `pre-commit` | Por instalar | Cada commit corre los gates; uno crítico en rojo **bloquea** |
| CI (`.github/workflows/`) | Por instalar | Cada push y PR corre los gates |

- Un gate crítico no se salta con `--no-verify`; se corrige o se justifica por escrito.
- Si cambian las reglas de los gates, `docs/GATES.md` se actualiza en el **mismo** commit.

## Límites

- **Siempre:** reportar con el comando y su salida real; actualizar `MEMORY.md` al cerrar fase.
- **Pregunta antes:** dependencias nuevas; cambios en el esquema de datos o en la API del core;
  tocar n8n, PostgreSQL preexistente u otros servicios del host; publicar.
- **Nunca:** secretos en el repo; `sudo`; comandos destructivos sin confirmación;
  recrear el contenedor n8n; deshabilitar un gate.

## Verificación

```powershell
docker compose config
pytest -q
```

- [ ] Tests en verde  - [ ] Sin secretos en el árbol git
- [ ] Criterios de la spec marcados con su verificación  - [ ] `MEMORY.md` actualizado

Si algo falla, **reporta el fallo exacto**; no lo reformules como "casi".
