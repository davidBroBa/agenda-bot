# MEMORY — Agenda inteligente y recordatorios por Telegram

Fase actual: **0 — documentación y diseño**. No hay código ni servicios desplegados.

## Decisiones tomadas (con su porqué)

- Reutilizar el n8n existente del servidor (petición del usuario). Se mitiga la
  fragilidad versionando los workflows como JSON en el repo.
- Recepción de Telegram por **long polling** con gateway propio: sin ingresos, sin
  `sudo`, funciona tras NAT.
- Lenguaje natural con **Ollama local `qwen3:8b`**: privado y sin coste.
- **Core y worker en Python 3.12** (FastAPI + psycopg + pytest).
- Motor de recordatorios **fuera de n8n**, en un worker propio, para que la ruta
  crítica no dependa de esa instancia.
- PostgreSQL 17 en contenedor propio, **sin puerto publicado**; no existe ninguno
  en el servidor todavía.

## Estado

- Hecho: diagnóstico del servidor, inventario de servicios, exploración de
  skills/agentes/recursos, diseño y documentación inicial.
- Verificado con: `srv.ps1 exec` (uname, docker ps, ss, curl Telegram, Caddyfile),
  `docker inspect n8n`.
- Pendiente: autenticación GitHub; aprobación final para Fase 1; crear
  `constitution.md`, `docs/SDD.md`, `docs/THREATS.md`, `docs/AI.md`.

## Próximos pasos

1. Resolver publicación en GitHub.
2. Aprobar el diseño y redactar la spec `001-bot-basico` (EARS) con el agente `planner`.
3. Implementar Fase 1 con test-first.
