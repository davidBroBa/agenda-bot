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
  skills/agentes/recursos, diseño y documentación.
- Documentos redactados: `constitution.md`, `docs/SDD.md`, `docs/THREATS.md`,
  `docs/AI.md`, `docs/NETWORK.md` y `docs/adr/0001..0004`.
- Repositorio publicado: `github.com/davidBroBa/agenda-bot` (público). El servidor
  empuja con la deploy key `~/.ssh/agenda_deploy`.
- Verificado con: `srv.ps1 exec` (uname, docker ps, ss, curl Telegram, Caddyfile),
  `docker inspect n8n`, `git push` + `git ls-files`.
- Diseño **aprobado**: supuestos S1-S6 del SDD §1.3 confirmados por David el
  2026-10-09.
- Creados el `AGENTS.md` del proyecto y `docs/HERRAMIENTAS.md` (skills y MCP
  recomendados) el 2026-10-10.
- Pendiente: decidir `docs/HERRAMIENTAS.md` §5; crear `docs/GATES.md` y la spec
  `001-bot-basico/`; arrancar la Fase 1.

## Próximos pasos

1. Decidir las 4 opciones de `docs/HERRAMIENTAS.md` §5 (config MCP, Postgres MCP,
   skills propias).
2. Redactar la spec `001-bot-basico` (EARS) con el agente `planner`.
3. Implementar Fase 1 con test-first.
