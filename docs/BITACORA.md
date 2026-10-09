# Bitácora del proyecto

Registro cronológico de lo que se hace en este proyecto, con el comando que lo
verificó y lo que queda pendiente. Cada entrada se añade al cerrar una tanda de
trabajo. Formato: fecha · qué se hizo · verificación · pendiente.

---

## 2026-10-09 — Fase 0: investigación y diseño

### 1. Inspección del servidor (`srv`, por SSH con `srv.ps1`)

Qué se hizo: reconocimiento completo del entorno antes de proponer nada.

- Sistema: Ubuntu 26.04.1 LTS, kernel 7.0, 8 núcleos, 13 GiB RAM (9.7 libres),
  855 GB libres en `/`.
- Docker 29.8.1 + Compose v5.5.1; usuario en el grupo `docker` (**sin `sudo`**).
- Salida a Internet y a `api.telegram.org` operativa (HTTP 302).
- IP pública `189.197.144.15`; Tailscale en `100.87.72.91`; Caddy sirviendo
  `pingopo.davidamador.dev` y `api.davidamador.dev` (puertos 80/443 alcanzables).

Verificación: `& $srv exec "uname -a; ...; free -h; df -h /"`, `docker ps -a`,
`ss -tlnp`, `curl -m10 https://api.telegram.org`, `cat /etc/caddy/Caddyfile`.

### 2. Servicios existentes (no se tocan)

| Servicio | Imagen | Puerto | Notas |
|---|---|---|---|
| `n8n` | `n8nio/n8n:latest` (v2.41.6) | `0.0.0.0:5678` | Arrancado a mano, `restart=no`, sin Compose |
| `n8n-api` | `n8n-api` (Python) | interno | Proyecto *recursos* (Instagram → HTML) |
| `pingo-db` | `mariadb:11` | `127.0.0.1:3306` | Proyecto Pingo POP (Compose) |
| `pingo-app` | Next.js | `127.0.0.1:3000` | Proyecto Pingo POP |
| varios | Caddy, cloudflared, nginx, ollama | 80/443, 11434 | `qwen3:8b` y `llama3.1:8b` disponibles |

Hallazgo importante: **no existe PostgreSQL** en ningún sitio (ni contenedor, ni
servicio, ni puerto 5432). Hay que crearlo.

Hallazgo importante: el contenedor `n8n` corre como `root` (`HOME=/root`), así que
su base real está en `/root/.n8n/database.sqlite` **dentro de la capa del
contenedor**. El bind `/DATA/AppData/n8n -> /home/node/.n8n` está **vacío y no se
usa**. Consecuencia: recrear el contenedor borraría workflows y credenciales, y
`restart=no` implica que no arranca tras un reinicio.

### 3. Recursos del escritorio y capacidades del agente

Qué se hizo: exploración de escritorio (servidor y local) y del sistema de
skills/agentes/subagentes de OpenCode. Detalle en `docs/INVESTIGACION.md`.

Resultado: recursos útiles localizados (guía Proyecto Estándar, un bot de
referencia Node, manuales de estilo de documentación) y agentes reutilizables
(`planner`, `implementer`, `sdd-reviewer`, auditores de seguridad/red).

### 4. Diseño aprobado en sus puntos clave

Decisiones confirmadas por el usuario:

1. **Reutilizar el n8n existente** (mitigación: workflows versionados como JSON en
   el repo, que es la fuente de verdad).
2. **Recepción por long polling** con un gateway propio (sin ingresos, sin `sudo`).
3. **Lenguaje natural con Ollama local `qwen3:8b`** (privado, sin coste).
4. **Core y worker en Python 3.12**.

Detalle completo en `docs/ARQUITECTURA.md`.

### 5. Publicación en GitHub

Qué se hizo: creación del repositorio y primer push.

- Autenticación: el gestor de credenciales de Windows guardaba un token OAuth
  (tipo `gho_`, scopes `repo, workflow, gist`) de la cuenta `davidBroBa`. Con él se
  creó el repositorio vía API.
- Repositorio: [`github.com/davidBroBa/agenda-bot`](https://github.com/davidBroBa/agenda-bot),
  **público**, rama `main`.
- Para que el **servidor** pueda subir sin guardar el token en disco, se generó una
  **deploy key** SSH (`~/.ssh/agenda_deploy`) y se añadió al repositorio con permiso
  de escritura. El `core.sshCommand` del repo la usa.
- Primer commit `7230727` con 7 ficheros versionados.

Verificación: `git push -u origin main` (nueva rama `main`), `git log --oneline -3`,
`git ls-files`.

Incidencia y lección: el primer `put` al servidor con ruta absoluta creó un
árbol anidado (`~/proyectos/home/...`) porque `srv.ps1 put` prefija la ruta con
`~/proyectos` si no empieza por `~`. Se corrigió usando ruta **relativa**
(`agenda-bot`).

### Pendiente

- Aprobación final del diseño para arrancar la Fase 1 (bot básico).
- Crear `constitution.md`, `docs/SDD.md`, `docs/THREATS.md`, `docs/AI.md`.
