# Herramientas — skills y MCP para agenda-bot

> Investigación de skills y servidores MCP útiles para este proyecto, con
> información consultada en internet (**2026-10-10**). Decide qué se instala; nada
> se instala sin autorización.

## 1. Entorno actual

- **OpenCode**: `v2.0.19` (cliente de escritorio, `OPENCODE_CLIENT=desktop`).
- **Skills globales**: `proyecto-estandar` (marco de trabajo) en
  `~/.config/opencode/skills`, y skills de diseño en `~/.opencode/skills`
  (`banner-design`, `brand`, `design`, `design-system`, `slides`, `ui-styling`,
  `ui-ux-pro-max`). Las de diseño **no aplican** aquí (la interfaz es Telegram).
- **Subagentes globales**: `coordinator`, `planner`, `implementer`,
  `sdd-reviewer`, `security-auditor`, `design-auditor`, `network-auditor`.
- **MCP** en `~/.config/opencode/opencode.json`: `refero`, `chrome-devtools`,
  `context7`.

## 2. Hallazgo: la config MCP está en formato V1

- El bloque `mcp` del `opencode.json` usa la forma **V1** (`mcp.<nombre>` +
  `enabled`). La documentación de **V2** indica que los servidores van en
  **`mcp.servers`** y que se usa `disabled` en vez de `enabled`.
  Fuente: <https://opencode.ai/v2/docs/mcp-servers/>.
- Comprobado con `opencode mcp list` en local → **"No MCP servers configured"**.
- **Pero** esta sesión sí usa `context7`. No encontré `context7` en ningún
  archivo de config local: el cliente de escritorio lo registra por su cuenta
  (gestión interna, no en disco accesible). No pude determinarlo con certeza.
- Consecuencia: **editar el `opencode.json` puede no ser la vía efectiva** en
  este setup. Para añadir MCP hay que usar la UI del desktop (`/mcps`) o migrar
  el archivo a la forma V2. **No lo cambio sin tu confirmación.**
- Nota de seguridad: `context7` guarda su token en **claro** dentro de
  `opencode.json`. Conviene moverlo a variable de entorno (`{env:...}`).

## 3. Skills: qué evalué y qué recomiendo

Ecosistema revisado (fuentes al final): colecciones comunitarias
`open-hax/opencode-skills`, `DamingDong/opencode-skills-hub` (285+ skills),
`Crandel/awesome-opencode-skills`, `dimkurilo/opencode-skills`, y el plugin npm
`opencode-skills-collection` (1000+ skills).

**Recomendación: NO instalar colecciones masivas.** Motivos:

- Son en su mayoría ports de Claude Code **sin auditar**; una skill puede traer
  scripts que el agente ejecuta → riesgo de cadena de suministro.
- Varios instaladores usan `curl ... | bash`, patrón **prohibido** por el
  `AGENTS.md` global.
- Decenas o cientos de skills contaminan el contexto y diluyen la selección.
- Este proyecto ya tiene un marco probado (`proyecto-estandar`) que cubre el flujo.

**Qué sí hacer:** crear **skills propias**, pequeñas y versionadas en el repo
(`skills/<id>/SKILL.md`), para lo repetitivo del proyecto:

| Skill propuesta | Para qué |
|---|---|
| `agenda-migraciones` | Crear/validar migraciones SQL y su reverso |
| `agenda-n8n-wf` | Exportar, versionar y validar los workflows JSON de n8n |
| `agenda-tests` | Patrón de tests con PostgreSQL efímero y timeouts controlados |

Opcional y de bajo riesgo (oficiales de Anthropic, publicadas en los hubs):
descargar solo **`skill-creator`** si quieres ayuda para redactar las skills
anteriores. No recomiendo instalar el resto.

## 4. MCP: recomendaciones

| Servidor | Para qué | Estado | Condiciones / riesgo | Cómo se añade |
|---|---|---|---|---|
| `context7` | Docs actuales de FastAPI, psycopg, dateutil | **Ya disponible** | Bajo. Mover token a env | Mantener |
| **Postgres MCP Pro** (`crystaldba/postgres-mcp`) | Ver esquema, `EXPLAIN`, salud, tuning; modo **read-only** | **Recomendado** | La DB no publica puerto → necesita **túnel SSH**; usar `--access-mode=restricted` | `docker run` o `uvx` |
| n8n (MCP de instancia) | Crear/ejecutar workflows por lenguaje natural | **No ahora** | Exige habilitar MCP **y reiniciar** el n8n existente (datos efímeros) → riesgo | Posponer |
| Telegram MCP (`chaindead/telegram-mcp`, `telegram-bot-mcp-server`) | Enviar/leer mensajes para pruebas manuales | Opcional | Un "userbot" usa tu **cuenta personal** (MTProto); el de bot usa token | Evaluar |
| Docker MCP Gateway | Catálogo de servidores aislados | No aplica | Requiere **Docker Desktop** (el servidor no lo tiene) | Descartar |
| `chrome-devtools`, `refero` | Navegador / diseño | No relevantes | Sin web UI en este proyecto | Considerar desactivar |

**Postgres MCP Pro** es el único que aporta valor claro ahora. Al ser read-only
por defecto y permitir `EXPLAIN`, encaja con la regla "no confiar, inspeccionar".
Como la base vive en el servidor sin puerto publicado, se conectaría por
**túnel SSH** (`ssh -L 5432:127.0.0.1:5432 ...`) apuntando a un usuario de solo
lectura, o corriendo el MCP en el propio servidor.

## 5. Decisiones abiertas (para ti)

1. ¿Migro el `opencode.json` a la forma V2 (`mcp.servers`) y saco el token de
   context7 a variable de entorno? (Recomendado, pero no toco tu config sin OK.)
2. ¿Instalo Postgres MCP Pro en modo restringido con túnel SSH? (Recomendado.)
3. ¿Quieres que cree las 3 skills propias del proyecto? (Recomendado.)
4. ¿Desactivamos `chrome-devtools` y `refero` para esta máquina? (Opcional.)

## 6. Fuentes (consultadas el 2026-10-10)

- OpenCode docs V2 — Agents: <https://opencode.ai/v2/docs/agents/>
- OpenCode docs V2 — Skills: <https://opencode.ai/v2/docs/skills/>
- OpenCode docs V2 — MCP servers: <https://opencode.ai/v2/docs/mcp-servers/>
- Postgres MCP Pro: <https://github.com/crystaldba/postgres-mcp>
- n8n MCP (built-in): <https://docs.n8n.io/connect/connect-to-n8n-mcp-server>
- Colecciones de skills: <https://github.com/Cramdel/awesome-opencode-skills>,
  <https://github.com/DamingDong/opencode-skills-hub>,
  <https://github.com/open-hax/opencode-skills>
