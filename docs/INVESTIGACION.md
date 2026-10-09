# Investigación del entorno

Recursos disponibles antes de empezar. Fecha: 2026-10-09.

## 1. Servidor `srv`

| | |
|---|---|
| Sistema | Ubuntu 26.04.1 LTS, kernel 7.0 |
| CPU / RAM / disco | 8 núcleos · 13 GiB (9.7 libres) · 855 GB libres |
| Docker / Compose | 29.8.1 / v5.5.1 (sin `sudo`, usuario en grupo `docker`) |
| Git | 2.53.0 |
| Node / Python | v22.23.3 / 3.14.4 (sin `pip3` en host; irrelevante, se usa Docker) |
| LLM local | Ollama con `qwen3:8b` y `llama3.1:8b` en `11434` |
| Red | Tailscale `100.87.72.91`; IP pública `189.197.144.15`; Caddy en 80/443 con `davidamador.dev` |
| Túneles | cloudflared (servicio root + túnel ad-hoc a `localhost:5678`) |
| Puertos libres relevantes | 5680 disponible |

Herramientas del wrapper `srv.ps1`: `check`, `init`, `exec`, `sh`, `put`, `get`,
`bg`, `log`, `ps`, `kill`, `gates`, `sudo`. Trampas conocidas en
`references/servidor.md` (CRLF, `~` entre comillas, `$HOME` interpolado por
PowerShell, `powershell -File` truncando argumentos).

## 2. Contenedores existentes (NO tocar)

`n8n`, `n8n-api`, `pingo-app`, `pingo-db`, `hermes`, `qbittorrent`, `radarr`,
`sonarr`, `prowlarr`, `jellyseerr`, `big-bear-nextcloud-ls`. Volúmenes: `n8n_data`,
`pingo-pop_pingo-db-data`, `pingo-pop_pingo-uploads`.

## 3. Skills, agentes y subagentes disponibles

### Skills (se cargan con la herramienta `skill`)

| Skill | Ubicación | ¿Sirve aquí? |
|---|---|---|
| `proyecto-estandar` | `~/.config/opencode/skills/` | **Sí**, es el marco obligatorio de trabajo |
| `context7-mcp` | `~/.agents/skills/` | Sí, para documentación de n8n/Telegram/Postgres |
| `design`, `design-system`, `ui-styling`, `ui-ux-pro-max`, `banner-design`, `brand`, `slides` | `~/.opencode/skills/` | No (no hay UI web; la interfaz es Telegram) |
| `opencode`, `report` | integradas | Puntuales (configuración de OpenCode, reporte de bugs) |

### Agentes (`~/.config/opencode/agents/`)

`coordinator`, `planner`, `implementer`, `sdd-reviewer`, `security-auditor`,
`network-auditor`, `design-auditor`.

### Subagentes invocables desde el orquestador

`explore` (exploración rápida), `general` (multi-paso), `implementer` (una tarea
SDD con test-first), `planner` (redacta spec/plan/tasks), `sdd-reviewer` (QA
read-only con veredicto).

### Comandos (`~/.config/opencode/commands/`)

`/proyecto`, `/sdd`, `/sdd-plan`, `/sdd-tasks`, `/sdd-implement`, `/sdd-validate`,
`/sdd-clarify`, `/sdd-status`, `/spec`, `/gates`, `/seguridad`, `/redes`,
`/diseno`, `/ia`, `/srv`.

**Uso previsto en este proyecto:** Fase B del marco `proyecto-estandar`. El agente
`planner` redacta cada `spec.md`/`plan.md`/`tasks.md`; el `implementer` implementa
una tarea ya aprobada con test-first; el `sdd-reviewer` valida RF por RF;
`security-auditor` y `network-auditor` revisan antes de exponer cualquier endpoint.

## 4. Recursos del escritorio

### Escritorio del servidor (`~/Desktop`)

- `GUIA-Proyecto-Estandar.md` / `.docx` — la guía del marco (ya disponible vía la skill).
- `Skills/` — symlink a `proyecto-estandar`.
- `Agentes/` — vacío.
- `Planeacion_Didactica_Ruta_del_Maiz_Mexico.docx` — no relacionado.

### Escritorio local (`C:\Users\brole\Desktop`)

- `GUIA-Proyecto-Estandar` (vía servidor), `OpenCode_Guia_Maestra.docx` — metodología.
- `manual-de-usuario-pingo-pop.md` y `Pingo POP - Documentación Técnica.docx` —
  **referencia de estilo de documentación** de un proyecto previo.
- `fb_bot_clean/` — bot en Node (`login.js`, `server.js`, sesión en JSON). Sirve
  como referencia de patrones de bot, aunque es de Facebook, no de Telegram.
- `Proyectos/` — fuentes de `Pingo-pop-main`, `portfolio`, `JS/Python`.
- `Coleccion de recursos - App.lnk` / `- Web.lnk` — accesos al proyecto *recursos*.

### Copia de trabajo en el servidor

`~/proyectos/` contiene además `resources/` (proyecto *recursos* con `db.py`,
`recursos.html`, `recursos.db`) y restos de trabajo previo de un bot de Telegram
para ese proyecto (`wf*.json`, `telegram-creds.json`, `n8n-python-api.py`). **Son
de otro proyecto**; no se reutilizan aquí. Este proyecto necesita su propio bot y
su propio token.

## 5. Conclusión

El entorno cubre todo lo necesario: Docker sin `sudo`, Ollama local para el
parsing, n8n existente para orquestar, Caddy/cloudflared como opción de ingreso y
el marco `proyecto-estandar` con sus agentes. El único hueco es **PostgreSQL**
(inexistente) y la **autenticación de GitHub** para publicar el repositorio.
