# Arquitectura

Diseño del asistente de agenda y recordatorios por Telegram. Estado: propuesto y
aprobado en sus decisiones clave; **pendiente de aprobación final para empezar a
implementar**.

## 1. Principios de diseño

- La ruta crítica de los recordatorios **no depende de n8n**: un worker propio
  reclama y envía. n8n orquesta (recibe, responde, resúmenes).
- Una sola capa toca el esquema de datos: el servicio `core`.
- Idempotencia real: claves únicas + transacciones para no duplicar envíos ni
  procesar dos veces el mismo mensaje de Telegram.
- Reproducibilidad: todo se levanta con Docker Compose; los workflows de n8n se
  versionan como JSON en el repositorio.
- Cero secretos en el repositorio: `.env` fuera de Git, `.env.example` dentro.

## 2. Diagrama

```
Telegram Cloud
   |  getUpdates (long polling)
   v
[ agenda-gateway ]  valida usuario autorizado -> dedupe update_id
   |  POST interno (secret) -> n8n existente (172.17.0.2:5678)
   v
[ n8n existente ]  01-ingress 02-commands 03-nl 04-dispatch(respaldo) 05-digest
   |  HTTP interno -> API del core
   v
[ agenda-core ]   FastAPI: unica capa que toca el esquema `agenda`
[ agenda-worker ] tick cada 30s: reclama con FOR UPDATE SKIP LOCKED, envia, registra
   |
   v
[ postgres:17 ]   base `agenda` (solo red interna, sin puerto publicado)
```

## 3. Componentes

| Componente | Rol |
|---|---|
| `agenda-gateway` | Long polling de Telegram, lista blanca de usuarios, dedupe por `update_id`, reenvío interno a n8n. |
| n8n (existente) | Orquestación: ingesta, comandos, lenguaje natural, digest, disparo de respaldo. |
| `agenda-core` | API HTTP con la lógica de dominio y el único acceso al esquema `agenda`. |
| `agenda-worker` | Bucle de recordatorios: reclama vencidos, envía, registra, recupera atrasados. |
| `postgres:17` | Persistencia transaccional. Solo red interna. Backups con `pg_dump`. |

## 4. Modelo de datos

- **`users`** — `telegram_user_id` UNIQUE, `timezone` (IANA), `quiet_start/quiet_end`,
  `digest_enabled/digest_time`, `enabled`. Lista blanca de acceso.
- **`priority_policies`** — `priority` PK (`normal|important|critical`),
  `lead_time` (anticipación), `repeat_interval` (frecuencia), `window` (ventana tras
  vencer), `max_notices`. Los tres niveles configurables.
- **`tasks`** — `id` uuid, `user_id` FK, `title`, `description`, `due_at timestamptz NULL`,
  `timezone`, `priority` FK, `status` (`pending|done|cancelled`), `rrule` (RFC-5545) +
  `rrule_until`, `parent_task_id` FK (instancias recurrentes), `version` (bloqueo
  optimista), timestamps.
- **`reminders`** — `task_id` FK `ON DELETE CASCADE`, `fire_at timestamptz`, `level`,
  `status` (`scheduled|claimed|sent|failed|skipped|cancelled`), `attempt_count`,
  `last_error`, `dedupe_key` UNIQUE. Índice parcial sobre `(status, fire_at)`.
- **`reminder_sends`** — libro antifraude: `idempotency_key` UNIQUE, `reminder_id` FK,
  `telegram_message_id`, `status`, `error`.
- **`bot_updates`** — `telegram_update_id` PK: deduplica reintentos de Telegram.
- **`audit_log`** y **`schema_migrations`**.

Reglas: al crear/editar una tarea con `due_at` se materializan sus `reminders` según
la política; `done`/`cancelled` cancelan los avisos pendientes; `snooze` desplaza
`fire_at`. Fechas siempre en UTC; hora local con zona IANA y `zoneinfo` (DST
correcto); recurrencia con `dateutil` + `TZID`.

Idempotencia de envío: se reclama en transacción
(`UPDATE reminders SET status='claimed' ... WHERE id IN (SELECT ... FOR UPDATE SKIP LOCKED) RETURNING *`),
se inserta en `reminder_sends` con `ON CONFLICT DO NOTHING` y solo si la inserción
prospera se envía. Un reaper devuelve a `scheduled` los `claimed` caducados.

## 5. Workflows de n8n (versionados en `n8n/workflows/`)

- `01-ingress` — recibe del gateway, valida usuario, deduplica.
- `02-commands` — `/add /list /today /week /done /snooze /cancel /prio` → API → respuesta.
- `03-nl` — texto libre → Ollama → JSON validado por esquema → confirmación.
- `04-dispatch` — Schedule cada minuto → `POST /tick` (respaldo por si cae el worker).
- `05-digest` — resumen diario/semanal según hora por usuario.

## 6. Decisiones y su porqué

| Decisión | Porqué | Alternativa descartada |
|---|---|---|
| Reutilizar el n8n existente | Petición del usuario; ahorra un contenedor | Stack n8n dedicado (más aislado y persistente) |
| Recepción por long polling | Sin ingresos, sin `sudo`, resistente a NAT | Webhook vía Caddy (más latencia baja, requiere DNS y `sudo`) |
| Ollama local `qwen3:8b` | Privado, sin coste, ya instalado | API externa (mejor calidad, coste y datos fuera) |
| Core/worker en Python | Mejor ecosistema de bots de Telegram; tests deterministas | Node/TypeScript (mismo runtime que n8n) |
| Motor fuera de n8n | La ruta crítica no depende de n8n | Todo en n8n (menos fiable con esta instancia) |

## 7. Riesgos

- El n8n reutilizado guarda datos en la capa del contenedor y `restart=no`.
  **Mitigación:** workflows versionados en el repo + script de reimportación.
- Token del bot y clave de n8n en `.env`; no se publican.
- Zona horaria y DST: se cubren con tests explícitos.
- Recuperación tras caída: el worker reprocesa atrasados dentro de su ventana.

## 8. Fases

| Fase | Contenido |
|---|---|
| 0 | Documentación: constitution, SDD, THREATS, AI, este documento, bitácora |
| 1 | Bot básico: Compose (postgres + gateway), migración `0001`, `/start`, `/help`, lista blanca, dedupe |
| 2 | Gestión de tareas: `0002`, API CRUD, comandos, recurrencia |
| 3 | Motor de recordatorios: `0003`, policies, worker, idempotencia, recuperación |
| 4 | Agenda interactiva y lenguaje natural |
| 5 | Fiabilidad: backups, restauración, healthchecks, gates |
