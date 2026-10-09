# Software Design Document — Agenda inteligente y recordatorios por Telegram

> **Se escribe antes de la primera línea de código.**
> Estructura basada en los 7 pasos. Secciones 6, 7 y 8 son obligatorias:
> "no aplica" es una respuesta válida; omitirlas no lo es.

---

## 1. Introducción

### 1.1 Propósito

Este sistema permite a una persona gestionar sus tareas y recibir recordatorios
fiables a través de un bot de Telegram. El problema que resuelve es que los
recordatorios de las apps de notas o del propio Telegram se pierden cuando el
usuario no está mirando el móvil, no admiten prioridades con política de aviso, y
no se recuperan si el servicio estuvo caído. Aquí la agenda es explícita, la
política de aviso es configurable y el motor de recordatorios es del propio
sistema.

### 1.2 Alcance

**Dentro de alcance:**
- Crear, consultar, editar, posponer, completar y cancelar tareas por Telegram.
- Comandos (`/add`, `/list`, `/today`, `/week`, `/done`, `/snooze`, `/cancel`, ...)
  y mensajes en lenguaje natural.
- Tres niveles de urgencia (normal, importante, crítico) con política propia:
  anticipación, frecuencia, ventana de repetición y máximo de avisos.
- Motor de recordatorios centralizado, idempotente y recuperable.
- Agenda diaria y semanal, pendientes y tareas vencidas.
- Tareas recurrentes (RFC 5545), horas de silencio y resúmenes diarios.
- Acceso restringido a identificadores de Telegram autorizados.
- Copias de seguridad, restauración y observabilidad básica.

**Fuera de alcance (explícitamente):**
- Aplicación web o móvil propia. La interfaz es Telegram.
- Multiusuario público / SaaS. Es una herramienta personal (lista blanca).
- Sincronización con Google Calendar, Outlook u otros calendarios. Se podrá
  evaluar como funcionalidad futura, no en el alcance inicial.
- Colaboración entre usuarios (compartir tareas). Fuera por simplicidad.
- Envío de correo o SMS. Telegram es el único canal.

### 1.3 Supuestos

Decisiones tomadas por omisión porque el usuario no las especificó. **Revisadas y
confirmadas con el usuario el 2026-10-09.**

| # | Supuesto | Impacto si es falso | Confirmado por |
|---|---|---|---|
| S1 | El proyecto vive en `~/proyectos/agenda-bot/` del servidor `srv` | Cambia rutas, Compose y scripts | David · 2026-10-09 |
| S2 | Zona horaria por defecto `America/Mexico_City` (configurable por usuario) | Avisos a horas equivocadas si el usuario está en otra zona | David · 2026-10-09 |
| S3 | Una sola instancia `postgres:17`, base `agenda` | Requiere otro contenedor o esquema | David · 2026-10-09 |
| S4 | El motor de recordatorios es un proceso propio, no n8n | Habría que rehacer la ruta crítica en n8n | David · 2026-10-09 |
| S5 | Un solo usuario (David) y volumen personal | El diseño simple podría no escalar | David · 2026-10-09 |
| S6 | El bot es de uso privado; el repo es público pero sin secretos | Exposición de datos si se descuida | David · 2026-10-09 |

> **Estado:** los seis supuestos quedaron confirmados por David el 2026-10-09.

### 1.4 Glosario

| Término | Definición |
|---|---|
| **Tarea** | Unidad de trabajo con título, fecha/hora opcional, prioridad y estado. |
| **Recordatorio** | Aviso materializado de una tarea, con hora de disparo y nivel. |
| **Política de prioridad** | Regla que define cuántos avisos y cada cuánto se emiten para un nivel. |
| **Tick** | Ejecución periódica del worker que reclama y envía los recordatorios vencidos. |
| **dedupe_key** | Clave única que evita que una tarea genere dos recordatorios idénticos. |
| **idempotency_key** | Clave única que evita enviar dos veces el mismo recordatorio. |
| **RRULE** | Regla de recurrencia de iCalendar (RFC 5545). |
| **Gateway** | Componente que hace long polling a Telegram y reenvía a n8n. |

---

## 2. Arquitectura del sistema

### 2.1 Visión general

```
                    [ Telegram Cloud ]
                            |  getUpdates (long polling)
                            v
                    +----------------+
                    | agenda-gateway |  lista blanca + dedupe update_id
                    +----------------+
                            |  POST interno (secreto compartido)
                            v
                    +----------------+        (contenedor existente, red del host)
                    |      n8n       |  01-ingress 02-commands 03-nl
                    +----------------+  04-dispatch(respaldo) 05-digest
                            |  HTTP interno
                            v
                    +----------------+
                    | agenda-core    |  FastAPI: unica capa que toca el esquema
                    +----------------+
                            ^
                            |  misma base de codigo (agenda.worker)
                    +----------------+
                    | agenda-worker  |  tick cada 30 s: reclama y envia
                    +----------------+
                            |
                            v
                    +----------------+
                    | postgres:17    |  red interna, sin puerto publicado
                    +----------------+

        agenda-core / agenda-worker --> ollama (host:11434) para lenguaje natural
```

### 2.2 Componentes principales

| Componente | Responsabilidad | Tecnología | Puerto |
|---|---|---|---|
| `agenda-gateway` | Long polling, lista blanca, dedupe de `update_id`, reenvío a n8n | Python (misma base) | interno |
| `n8n` (existente) | Orquestación: ingesta, comandos, NL, digest, disparo de respaldo, respuestas finales | n8n v2.41 | 5678 (host) |
| `agenda-core` | API de dominio y **único** acceso al esquema `agenda` | FastAPI + psycopg | 8080 (interno) |
| `agenda-worker` | Motor de recordatorios: reclama vencidos, envía, registra | Python (misma base) | interno |
| `postgres:17` | Persistencia transaccional | PostgreSQL 17 | 5432 (interno) |
| `ollama` (existente) | Parsing de lenguaje natural | Ollama `qwen3:8b` | 11434 (host) |

### 2.3 Flujo de datos

**Alta de una tarea por comando:**
1. El usuario envía `/add ...` a Telegram.
2. `agenda-gateway` lo recibe por long polling, comprueba la lista blanca,
   inserta `telegram_update_id` en `bot_updates` (dedupe) y lo reenvía a n8n con
   un secreto compartido.
3. n8n (`02-commands`) llama a `POST /tasks` del `core`.
4. El `core` valida con Pydantic en el borde, resuelve la política de prioridad,
   inserta la tarea y **materializa** sus `reminders` en una transacción.
5. El `core` responde; n8n contesta por Telegram.

**Disparo de un recordatorio:**
1. El `worker` (cada 30 s) selecciona recordatorios `scheduled` con `fire_at <= now()`
   usando `SELECT ... FOR UPDATE SKIP LOCKED`.
2. En la misma transacción inserta en `reminder_sends` con `ON CONFLICT DO NOTHING`.
   Si la clave ya existía, salta (ya se envió).
3. Fuera de la transacción, envía el mensaje por la API de Telegram.
4. Registra el resultado (`sent`/`failed`) y el `telegram_message_id`.

Cada salto valida su entrada: el gateway valida el `Update` de Telegram con un
schema; el core valida el cuerpo HTTP con Pydantic; el worker solo toca datos que
él mismo ha reclamado.

### 2.4 Decisiones transversales

| Tema | Decisión | Por qué |
|---|---|---|
| Autenticación | Lista blanca de `telegram_user_id`; secreto compartido entre gateway y n8n; n8n y core en red interna | No hay login de usuario; el perímetro es la lista blanca y la red |
| Autorización | Cada operación filtra por `user_id` del remitente | Evita acceso a tareas de otro usuario |
| Manejo de errores | Excepciones tipadas; el envío fallido reintenta con tope; nada de `except` vacío | Fiabilidad y diagnóstico |
| Logging | Estructurado (JSON), con `update_id`/`task_id`; **nunca** secretos ni contenido sensible completo | Trazabilidad sin filtrar datos |
| Configuración | Todo por variables de entorno; `.env` fuera de Git, `.env.example` dentro | Sin secretos en el repo |
| Fechas | Almacenar en UTC (`timestamptz`) + zona IANA del usuario; `zoneinfo` para conversión | DST y viajes correctos |

---

## 3. Diseño de componentes

### 3.1 agenda.domain

- **Responsabilidad:** lógica de negocio pura: crear/editar tareas, calcular la
  próxima ocurrencia de una recurrencia, materializar recordatorios desde una
  política, decidir si un aviso cae dentro de la ventana.
- **Interfaz:**
  - `create_task(cmd: CreateTaskCommand, now: datetime) -> Task`
  - `materialize_reminders(task, policy, now) -> list[ReminderSpec]`
  - `next_occurrence(task, after: datetime) -> datetime | None`
  - `is_within_window(reminder, now) -> bool`
- **Dependencias:** `agenda.lib` (fechas), `agenda.config`. **No** importa FastAPI
  ni psycopg.
- **No responsable de:** persistencia, HTTP, envío a Telegram.

### 3.2 agenda.api

- **Responsabilidad:** exponer la API HTTP y validar el borde con Pydantic.
- **Interfaz:** routers `/tasks`, `/today`, `/week`, `/tick`, `/healthz`.
- **Dependencias:** `agenda.domain`, `agenda.db`.
- **No responsable de:** contener reglas de negocio (solo orquesta y traduce).

### 3.3 agenda.worker

- **Responsabilidad:** bucle de recordatorios: reclamar, enviar y registrar.
- **Interfaz:** `run_once(now) -> TickResult`, `main()`.
- **Dependencias:** `agenda.db`, `agenda.notify` (cliente Telegram).
- **No responsable de:** parsear lenguaje natural ni atender HTTP.

### 3.4 agenda.gateway

- **Responsabilidad:** long polling de Telegram, lista blanca, dedupe por
  `update_id`, reenvío a n8n.
- **Interfaz:** `poll_forever()`, `handle_update(update) -> None`.
- **Dependencias:** `agenda.db`, `agenda.notify`.
- **No responsable de:** lógica de tareas.

### 3.5 agenda.nlp

- **Responsabilidad:** convertir texto libre en una intención estructurada usando
  Ollama, con salida validada.
- **Interfaz:** `parse_intent(text, timezone, now) -> ParsedIntent | None`.
- **Dependencias:** cliente HTTP a Ollama, `agenda.lib` (fechas).
- **No responsable de:** ejecutar la acción; solo propone y se valida después.

---

## 4. Diseño de datos

### 4.1 Modelo de datos

| Entidad | Campo | Tipo | Null | PK/FK | Restricción |
|---|---|---|---|---|---|
| `users` | `id` | uuid | no | PK | |
| | `telegram_user_id` | bigint | no | | UNIQUE |
| | `timezone` | text | no | | IANA válida |
| | `quiet_start` / `quiet_end` | time | sí | | horas de silencio |
| | `digest_enabled` / `digest_time` | bool / time | no/sí | | |
| | `enabled` | bool | no | | default true |
| `priority_policies` | `priority` | text | no | PK | `normal|important|critical` |
| | `lead_time` | interval | no | | anticipación |
| | `repeat_interval` | interval | no | | frecuencia |
| | `window` | interval | no | | ventana tras vencer |
| | `max_notices` | int | no | | ≥ 1 |
| `tasks` | `id` | uuid | no | PK | |
| | `user_id` | uuid | no | FK users | ON DELETE CASCADE |
| | `title` | text | no | | 1..200 |
| | `description` | text | sí | | |
| | `due_at` | timestamptz | sí | | `NULL` = sin vencimiento |
| | `timezone` | text | no | | copia de la del usuario al crear |
| | `priority` | text | no | FK policies | |
| | `status` | text | no | | `pending|done|cancelled` |
| | `rrule` / `rrule_until` | text / timestamptz | sí | | recurrencia |
| | `parent_task_id` | uuid | sí | FK tasks | instancia recurrente |
| | `version` | int | no | | bloqueo optimista |
| | `created_at` / `updated_at` | timestamptz | no | | |
| `reminders` | `id` | uuid | no | PK | |
| | `task_id` | uuid | no | FK tasks | ON DELETE CASCADE |
| | `fire_at` | timestamptz | no | | |
| | `level` | int | no | | nº de aviso |
| | `status` | text | no | | `scheduled|claimed|sent|failed|skipped|cancelled` |
| | `attempt_count` | int | no | | default 0 |
| | `last_error` | text | sí | | |
| | `dedupe_key` | text | no | | UNIQUE |
| `reminder_sends` | `id` | uuid | no | PK | |
| | `reminder_id` | uuid | no | FK reminders | |
| | `idempotency_key` | text | no | | UNIQUE |
| | `telegram_message_id` | bigint | sí | | |
| | `status` | text | no | | `sent|failed` |
| | `error` | text | sí | | |
| | `sent_at` | timestamptz | sí | | |
| `bot_updates` | `telegram_update_id` | bigint | no | PK | dedupe de Telegram |
| | `received_at` | timestamptz | no | | |
| `audit_log` | `id` | bigserial | no | PK | |
| | `actor` | text | no | | user id o sistema |
| | `action` | text | no | | |
| | `entity` / `entity_id` | text / uuid | sí | | |
| | `payload` | jsonb | sí | | sin secretos |
| | `created_at` | timestamptz | no | | |
| `schema_migrations` | `version` | text | no | PK | |
| | `applied_at` | timestamptz | no | | |

Índices: parcial `reminders(status, fire_at) WHERE status='scheduled'`;
`tasks(user_id, status, due_at)`; `reminders(task_id)`.

### 4.2 Esquema

```sql
CREATE TABLE users (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  telegram_user_id bigint NOT NULL UNIQUE,
  timezone         text   NOT NULL,
  quiet_start      time,
  quiet_end        time,
  digest_enabled   boolean NOT NULL DEFAULT true,
  digest_time      time,
  enabled          boolean NOT NULL DEFAULT true
);

CREATE TABLE priority_policies (
  priority        text PRIMARY KEY CHECK (priority IN ('normal','important','critical')),
  lead_time       interval NOT NULL,
  repeat_interval interval NOT NULL,
  window          interval NOT NULL,
  max_notices     int NOT NULL CHECK (max_notices >= 1)
);

CREATE TABLE tasks (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title          text NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
  description    text,
  due_at         timestamptz,
  timezone       text NOT NULL,
  priority       text NOT NULL REFERENCES priority_policies(priority),
  status         text NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','done','cancelled')),
  rrule          text,
  rrule_until    timestamptz,
  parent_task_id uuid REFERENCES tasks(id) ON DELETE CASCADE,
  version        int NOT NULL DEFAULT 1,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE reminders (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id       uuid NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  fire_at       timestamptz NOT NULL,
  level         int NOT NULL,
  status        text NOT NULL DEFAULT 'scheduled'
                CHECK (status IN ('scheduled','claimed','sent','failed','skipped','cancelled')),
  attempt_count int NOT NULL DEFAULT 0,
  last_error    text,
  dedupe_key    text NOT NULL UNIQUE
);
CREATE INDEX reminders_due_idx ON reminders (status, fire_at) WHERE status = 'scheduled';

CREATE TABLE reminder_sends (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  reminder_id         uuid NOT NULL REFERENCES reminders(id) ON DELETE CASCADE,
  idempotency_key     text NOT NULL UNIQUE,
  telegram_message_id bigint,
  status              text NOT NULL CHECK (status IN ('sent','failed')),
  error               text,
  sent_at             timestamptz
);

CREATE TABLE bot_updates (
  telegram_update_id bigint PRIMARY KEY,
  received_at        timestamptz NOT NULL DEFAULT now()
);
```

### 4.3 Migraciones y versionado

| Herramienta | Estrategia |
|---|---|
| SQL propio (`db/migrations/NNNN_*.sql`) aplicado por un runner mínimo | Migraciones versionadas, reversibles cuando sea posible, registradas en `schema_migrations` |

### 4.4 Retención y borrado

| Dato | Retención | Cómo se borra |
|---|---|---|
| `tasks` y `reminders` | Hasta que el usuario los borre o cancele | Borrado lógico (`status`) o `DELETE` con cascada |
| `reminder_sends` | 90 días | Job de limpieza |
| `bot_updates` | 30 días | Job de limpieza (solo sirve para dedupe reciente) |
| `audit_log` | 1 año | Job de limpieza |

---

## 5. Diseño de interfaces

### 5.1 API

| Método | Ruta | Descripción | Auth | Errores |
|---|---|---|---|---|
| `POST` | `/tasks` | Crear tarea (y materializar recordatorios) | interna | 400, 422, 500 |
| `GET` | `/tasks` | Listar con filtros (estado, rango, prioridad) | interna | 400, 500 |
| `GET` | `/tasks/{id}` | Detalle | interna | 404, 500 |
| `PATCH` | `/tasks/{id}` | Editar / posponer / cambiar prioridad (bloqueo optimista) | interna | 404, 409, 422 |
| `POST` | `/tasks/{id}/done` | Completar y cancelar avisos | interna | 404, 409 |
| `POST` | `/tasks/{id}/cancel` | Cancelar y cancelar avisos | interna | 404, 409 |
| `GET` | `/agenda/today` | Agenda del día en la zona del usuario | interna | 400, 500 |
| `GET` | `/agenda/week` | Agenda de la semana | interna | 400, 500 |
| `POST` | `/tick` | Fuerza un tick del motor (respaldo desde n8n) | interna | 500 |
| `GET` | `/healthz` | Salud del core y de la base | no | 503 |

"interna" = solo accesible desde la red Docker; no se publica en el host.

### 5.2 Interfaz de usuario

| Pantalla | Propósito | Estados (carga/vacío/error) |
|---|---|---|
| Chat de Telegram | Todo el uso: comandos y lenguaje natural | No aplica carga; si el bot no entiende, responde con ayuda y ejemplo |

### 5.3 Eventos y mensajes

| Evento | Emisor | Consumidores | Payload |
|---|---|---|---|
| `telegram.update` | gateway | n8n (`01-ingress`) | `Update` de Telegram + `dedupe` |
| `task.created` / `task.updated` | core | (interno) | id de tarea |
| `reminder.due` | worker | Telegram | texto del aviso |

---

## 6. Seguridad

### 6.1 Autenticación y autorización

| Aspecto | Decisión |
|---|---|
| Método | Lista blanca de `telegram_user_id`; secreto compartido gateway↔n8n |
| Expiración de sesión | No aplica (sin sesiones de usuario) |
| Modelo de permisos | Cada operación filtra por el `user_id` del remitente |

### 6.2 Modelo de amenazas

Referencia: [`THREATS.md`](./THREATS.md) — documento completo.

### 6.3 Protección de datos

| Dato sensible | Cómo se protege |
|---|---|
| Token del bot | Variable de entorno; nunca en logs ni en el repo |
| Secreto gateway↔n8n | Variable de entorno; red interna |
| Contenido de las tareas | Solo en PostgreSQL interno; el modelo local no envía nada a terceros |
| Identificadores de Telegram | Solo en la base interna; no se publican |

---

## 7. Red y despliegue

### 7.1 Topología

Referencia: [`NETWORK.md`](./NETWORK.md)

### 7.2 CI/CD

| Etapa | Trigger | Comando |
|---|---|---|
| Typecheck | push/PR | `mypy --strict src` |
| Lint | push/PR | `ruff check .` |
| Tests | push/PR | `pytest` |
| Auditoría | push/PR | `pip-audit` |

> Se añadirán en `docs/GATES.md` y `.github/workflows/`; en el servidor no hay
> `pwsh`, así que los gates se ejecutan en local sobre el código descargado.

### 7.3 Monitoreo

| Qué | Herramienta | Dónde se alerta |
|---|---|---|
| Errores | logs JSON + `healthz` | log del contenedor |
| Latencia p95 del tick | métrica del worker | log |
| Recordatorios atrasados | consulta a `reminders` | healthcheck |

---

## 8. IA

> **Presente.** El sistema usa un modelo local para interpretar lenguaje natural.

### 8.1 Modelo y proveedor

Ollama local ejecutando `qwen3:8b`. Se elige por privacidad (los datos no salen
del servidor) y por coste cero. Alternativas evaluadas en `docs/AI.md`.

### 8.2 Diseño de prompts

Plantillas versionadas en `prompts/`. Estructura: rol, contexto (delimitado y
marcado como datos), tarea, formato JSON exacto, restricciones y ejemplos.

### 8.3 Evaluación

Dataset en `evals/dataset.jsonl`, mínimo 30 casos (camino feliz, ambiguos, datos
faltantes, entrada hostil). Umbral: ≥ 90% de aciertos. Se ejecutará en local.

### 8.4 Guardrails

Validación de entrada (límite de tokens), salida validada con Pydantic, y
**human-in-the-loop**: el bot propone y pide confirmación antes de crear o
modificar. La autorización siempre en el código, nunca en el prompt.

### 8.5 Coste

| Métrica | Valor |
|---|---|
| Tokens por llamada | acotado por `max_tokens` |
| Coste por usuario al mes | 0 (modelo local) |
| Presupuesto máximo mensual | 0 |
| Alerta configurada en | No aplica |

### 8.6 Privacidad

Al modelo solo llega el texto que el usuario escribió al bot y las opciones
necesarias (zona horaria, "ahora"). El modelo corre en local; el proveedor no es
un tercero y no entrena con los datos.

### 8.7 Alternativas sin IA

Se evaluó un parser de reglas para las frases más comunes. Se descartó como
único mecanismo porque no cubre la variabilidad del lenguaje, pero **sí** se usa
como primera capa (comandos y patrones simples) y el modelo solo actúa cuando
la regla no basta. Ver `docs/AI.md` §11.

### 8.8 No aplica — justificación

No aplica: la IA sí se usa. Ver secciones anteriores.

---

## 9. Rendimiento y escalabilidad

| Métrica | Objetivo | Cómo se mide |
|---|---|---|
| Latencia p95 del tick | < 200 ms (sin contar llamadas a Telegram) | log del worker |
| Latencia de respuesta del bot | < 3 s (comandos), < 8 s (lenguaje natural) | log |
| Usuarios concurrentes | < 10 | supuesto S5 |
| Throughput | decenas de tareas/día | uso personal |

---

## 10. Plan de implementación

| Fase | Entregable | Depende de |
|---|---|---|
| 0 | Documentación: constitution, SDD, THREATS, AI, NETWORK, ARQUITECTURA, bitácora | — |
| 1 | Bot básico: Compose (postgres + gateway), migración `0001`, `/start`, `/help`, lista blanca, dedupe | 0 |
| 2 | Gestión de tareas: `0002`, API CRUD, comandos, recurrencia | 1 |
| 3 | Motor de recordatorios: `0003`, políticas, worker, idempotencia, recuperación | 2 |
| 4 | Agenda interactiva y lenguaje natural | 3 |
| 5 | Fiabilidad: backups, restauración, healthchecks, gates | 4 |

---

## 11. Riesgos

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| El n8n existente pierde sus datos (capa del contenedor, `restart=no`) | Media | Alto | Workflows versionados en el repo + script de reimportación |
| Fuga del token del bot | Baja | Alto | Variables de entorno, sin logs, repo con barrido de secretos |
| DST / zona horaria mal aplicada | Media | Medio | `zoneinfo` + tests de cambio de hora |
| Recordatorios duplicados o perdidos | Media | Alto | Claves únicas + transacciones + reaper |
| Ollama no disponible | Media | Bajo | Primera capa determinista; si el modelo cae, se pide reformular |

---

## 12. Apéndices

- Glosario: §1.4.
- Referencias: `docs/ARQUITECTURA.md`, `docs/THREATS.md`, `docs/AI.md`, `docs/NETWORK.md`.
- Diagramas adicionales: `docs/NETWORK.md`.

---

## Aprobación

| Rol | Nombre | Fecha | Decisión |
|---|---|---|---|
| Responsable del proyecto | David | 2026-10-09 | Aprobado (supuestos S1-S6 confirmados) |
