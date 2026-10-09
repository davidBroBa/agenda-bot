# Agenda inteligente y recordatorios por Telegram

Asistente personal para gestionar tareas y recibir recordatorios fiables a través
de un bot de Telegram. La automatización se apoya en un n8n, los datos viven en
PostgreSQL y el motor de recordatorios es un servicio propio en Python.

- **Estado:** Fase 0 — documentación y diseño. Sin código ni servicios desplegados.
- **Repositorio:** [`github.com/davidBroBa/agenda-bot`](https://github.com/davidBroBa/agenda-bot) (público, rama `main`).
- **Servidor de trabajo:** alias SSH `srv` (Ubuntu 26.04, Docker).
- **Copia de trabajo:** `~/proyectos/agenda-bot/` en `srv`.

## Capacidades objetivo

- Crear, consultar, editar, posponer, completar y cancelar tareas desde Telegram.
- Comandos y mensajes en lenguaje natural.
- Tres niveles de urgencia (normal, importante, crítico) con política configurable
  de anticipación, frecuencia, ventana de repetición y máximo de avisos.
- Motor de recordatorios centralizado, idempotente y recuperable tras caídas.
- Agenda diaria y semanal, pendientes y tareas vencidas.
- Tareas recurrentes, horas de silencio y resúmenes diarios.
- Acceso restringido a identificadores de Telegram autorizados.

## Stack

| Capa | Tecnología |
|---|---|
| Automatización | n8n (se reutiliza el existente del servidor) |
| Base de datos | PostgreSQL 17 (contenedor propio, sin exposición pública) |
| Interfaz | Telegram Bot API (recepción por long polling) |
| Motor / core | Python 3.12 + FastAPI + worker |
| Lenguaje natural | Ollama local `qwen3:8b` (salida JSON validada) |
| Despliegue | Docker Compose |
| Control de versiones | Git (sin secretos) |

## Estructura prevista

```
agenda-bot/
  README.md  AGENTS.md  MEMORY.md  constitution.md
  docker-compose.yml  .env.example  .gitignore
  docs/      INVESTIGACION.md  ARQUITECTURA.md  BITACORA.md  SDD.md  THREATS.md ...
  specs/     001-bot-basico/ 002-gestion-tareas/ ... (spec, plan, tasks)
  core/      src/agenda/  tests/
  db/        migrations/
  n8n/       workflows/
  ops/       backup.sh  restore.sh  healthcheck.sh
```

## Documentación

- `docs/INVESTIGACION.md` — entorno del servidor y recursos/skills/agentes explorados.
- `docs/ARQUITECTURA.md` — arquitectura, modelo de datos, workflows y decisiones.
- `docs/BITACORA.md` — registro cronológico de cada acción realizada en el proyecto.
