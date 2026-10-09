# Modelo de Amenazas — Agenda inteligente y recordatorios por Telegram

> **Este documento se escribe ANTES de implementar.**
> Si lo estás completando después del código, el ejercicio ya falló: úsalo al
> menos para detectar lo que se te pasó y dejarlo registrado.

---

## 1. Alcance del análisis

- **Sistema analizado:** Agenda inteligente y recordatorios por Telegram, versión 0 (diseño).
- **Fecha:** 2026-10-09
- **Autor:** Agente de diseño (revisión pendiente de David).
- **Fuera de alcance:** los servicios preexistentes del servidor (`pingo-app`,
  Nextcloud, *arr, qbittorrent, etc.) **no** se analizan en profundidad; solo se
  consideran como superficie de riesgo compartida (el host). Tampoco se analiza
  la seguridad interna de Telegram ni de n8n como producto; se analiza cómo los
  usamos y los ajustes que dependen de nosotros.

## 2. Activos

| # | Activo | Tipo | Criticidad | Dónde vive |
|---|---|---|---|---|
| A1 | Token del bot de Telegram | Credencial | Alta | `.env` → entorno del contenedor |
| A2 | Base de datos `agenda` (agenda personal) | Datos personales | Alta | `postgres:17` (red interna) |
| A3 | Secreto compartido gateway ↔ n8n | Credencial | Media | `.env` de ambos |
| A4 | Acceso al bot (quién puede usarlo) | Control de acceso | Alta | Tabla `users` + lista blanca |
| A5 | El host y sus demás contenedores | Infraestructura | Crítica | Servidor `srv` |
| A6 | Workflows y credenciales de n8n | Configuración/Credenciales | Alta | n8n (capa del contenedor) |
| A7 | Salida del modelo de lenguaje | Integridad | Media | Ollama local |
| A8 | Copias de seguridad | Datos | Alta | `ops/` → disco del servidor |

## 3. Puntos de entrada

| # | Punto | Quién lo invoca | Autenticado |
|---|---|---|---|
| E1 | API de Telegram (`getUpdates`) | Telegram Cloud | Sí (token del bot) |
| E2 | `POST` interno gateway → n8n | agenda-gateway | Secreto compartido |
| E3 | HTTP interno n8n → core | n8n | Red interna |
| E4 | `POST /tick`, API de tareas del core | n8n / worker | Red interna |
| E5 | API de Ollama (host `:11434`) | core / n8n | No (local) |
| E6 | SSH del servidor | David | Sí (clave) |
| E7 | Caddy (80/443) y cloudflared | Internet | N/A (sirve otros dominios) |
| E8 | UI de n8n (host `:5678`) | David / Internet | Sí (login de n8n), **ligado a 0.0.0.0** |

## 4. Fronteras de confianza

```
[Internet: NO confiable]
   |  (1) Telegram getUpdates (salida)   (2) :5678 n8n   (3) :80/:443 Caddy
   v
[Servidor srv: confiable parcial]
   |-- contenedores existentes (n8n, pingo, *arr, ollama, ...)   <- NO son nuestros
   |-- (4) red Docker del proyecto agenda
   |        |
   |        v
   |   [agenda-gateway / core / worker]  <- confiable (nuestro codigo)
   |        |
   |        v
   |   [postgres:17]  <- confiable; SIN puerto publicado
   v
[Host / disco]  <- backups en claro (pendiente de cifrado)
```

Controles de frontera: el token del bot autentica (1); el secreto compartido
autentica (4); el aislamiento de red protege la base; nada del proyecto se
publica en el host. **Puntos abiertos heredados:** (2) y (5) ya existían.

## 5. Amenazas por activo (STRIDE)

### A1 — Token del bot de Telegram

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **S** | Suplantación | Alguien obtiene el token y actúa como el bot | Alto | Baja | Solo en `.env`; nunca en logs ni repo; rotación con BotFather | Bajo |
| **T** | Alteración | Se modifica el token en el `.env` | Medio | Baja | Permisos del fichero; cambios revisados | Bajo |
| **R** | Negación | — | Bajo | Baja | Bitácora y auditoría | Bajo |
| **I** | Filtración | El token aparece en un log o en el repo | Alto | Media | Filtro de logs; barrido de secretos; `.gitignore` | Medio |
| **D** | Denegación | Abuso de la API hasta el límite | Medio | Baja | Long polling (no expone webhook); control de errores | Bajo |
| **E** | Escalada | Con el token, controlar el bot | Alto | Baja | No hay más privilegios que el bot en sí | Bajo |

### A2 — Base de datos `agenda`

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **S** | Suplantación | Acceso a Postgres suplantando al core | Alto | Baja | Sin puerto publicado; credenciales por entorno | Bajo |
| **T** | Alteración | Modificar tareas sin autorización | Alto | Baja | Autorización por `user_id`; bloqueo optimista; auditoría | Bajo |
| **R** | Negación | Negar haber borrado una tarea | Medio | Baja | `audit_log` con actor y fecha | Bajo |
| **I** | Filtración | Consultar tareas de otro usuario (IDOR) | Alto | Media | Filtrado por `user_id` en **cada** consulta; test de IDOR | Bajo |
| **D** | Denegación | Inundar la base hasta tumbarla | Medio | Baja | Volumen personal; límites de conexiones | Bajo |
| **E** | Escalada | Usar SQL para salir del contenedor | Alto | Baja | Consultas parametrizadas; usuario no-root del contenedor | Bajo |

### A3 — Secreto gateway ↔ n8n

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **S** | Suplantación | Inyectar updates falsos a n8n | Alto | Baja | Secreto compartido; red interna | Bajo |
| **I** | Filtración | El secreto se filtra en logs | Medio | Baja | No loggear cabeceras; `.env` | Bajo |

### A4 — Acceso al bot

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **S** | Suplantación | Un desconocido escribe al bot | Medio | Alta | Lista blanca; se ignora (o se responde educadamente) sin autorizar | Bajo |
| **E** | Escalada | Un usuario autorizado intenta ver datos de otro | Alto | Baja | Cada operación usa el `user_id` del **remitente**, no del payload | Bajo |
| **D** | Denegación | Spam de mensajes al bot | Medio | Media | Rate limit por usuario; ignorar no autorizados | Bajo |

### A5 — El host y sus demás contenedores

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **S/T/I/D/E** | Comprometer el host desde nuestro stack | Un fallo en nuestro contenedor permite salir al host y tocar otros servicios | Crítico | Baja | Sin `sudo`; contenedores sin `privileged`; `no-new-privileges`; usuario no-root | Medio |
| **I** | Exposición de servicios existentes | n8n (`:5678`) y Ollama (`:11434`) ligados a `0.0.0.0` | Alto | Media | **Recomendación:** ligar a `127.0.0.1`; firewall; no es nuestro contenedor, se reporta | **Medio (aceptado)** |

### A6 — Workflows y credenciales de n8n

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **T** | Alteración | Un cambio rompe los workflows | Medio | Media | Workflows versionados en el repo; reimportables | Bajo |
| **I** | Filtración | Credencial del bot dentro de n8n | Alto | Baja | Se crea una credencial **nueva**; el export no va al repo | Medio |
| **D** | Denegación | Recrear el contenedor borra los datos | Alto | Media | **No recrear**; nota en la bitácora; backup de n8n antes de tocar | Medio |

### A7 — Salida del modelo de lenguaje

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **T** | Alteración | Prompt injection desde el texto del usuario | Medio | Media | El texto del usuario va como **dato**, nunca concatenado a instrucciones; salida validada con Pydantic; acciones con confirmación | Bajo |
| **I** | Filtración | Mandar datos a un tercero | Bajo | Baja | Modelo **local**; no hay tercero | Bajo |
| **D** | Denegación | El modelo cuelga el bot | Medio | Media | Timeout + reintento; primera capa determinista | Bajo |

### A8 — Copias de seguridad

| | Amenaza | Escenario concreto | Impacto | Prob. | Controles | Riesgo residual |
|---|---|---|---|---|---|---|
| **I** | Filtración | Un dump con datos queda en claro | Alto | Media | Permisos restrictivos; **pendiente** cifrar el dump | Medio |

## 6. Top 10 de mitigaciones priorizadas

| Prioridad | Mitigación | Amenaza que cubre | Responsable | Estado |
|---|---|---|---|---|
| 1 | Filtrar **toda** consulta por el `user_id` del remitente + test explícito de IDOR | I (A2, A4) | Desarrollo | Pendiente |
| 2 | Claves únicas (`dedupe_key`, `idempotency_key`) y transacciones en el envío | Duplicados/pérdidas | Desarrollo | Pendiente |
| 3 | Barrido de secretos (`gitleaks`) en cada commit; `.env` ignorado | I (A1, A3, A8) | Desarrollo | Pendiente |
| 4 | Contenedores sin `privileged`, `no-new-privileges`, usuario no-root | E (A5) | Desarrollo | Pendiente |
| 5 | Postgres y core **sin puerto publicado** en el host | S/I (A2, A4) | Desarrollo | Pendiente |
| 6 | Reportar y recomendar ligar n8n y Ollama a `127.0.0.1` | I (A5) | David (servicios existentes) | Pendiente |
| 7 | Salida del modelo validada con Pydantic + confirmación (human-in-the-loop) | T (A7, A4) | Desarrollo | Pendiente |
| 8 | Timeout + reintento + reaper que devuelve `claimed` caducados | D (A2, A7) | Desarrollo | Pendiente |
| 9 | `audit_log` con actor y fecha, sin secretos | R (A2) | Desarrollo | Pendiente |
| 10 | No recrear el contenedor n8n; workflows versionados en el repo | D (A6) | Desarrollo | Pendiente |

## 7. Riesgos aceptados

Riesgos que se deciden **no** corregir (o no ahora). Cada uno necesita justificación.

| Riesgo | Justificación | Revisado por | Fecha |
|---|---|---|---|
| n8n y Ollama escuchan en `0.0.0.0` | Son contenedores **preexistentes**; modificarlos está fuera del encargo ("no eliminar ni reemplazar servicios"). Se reporta y se recomienda el cambio. | David | 2026-10-09 |
| n8n guarda datos en la capa del contenedor y `restart=no` | No se puede cambiar sin recrear el contenedor (pérdida de datos) y sin `sudo`/decisión del usuario. Se mitiga versionando los workflows. | David | 2026-10-09 |
| Backups sin cifrar (inicial) | Primera versión; se cifrará si el volumen o la sensibilidad lo exige. Permisos restrictivos desde el inicio. | David | 2026-10-09 |
| El repo es público | Decisión explícita del usuario. No habrá secretos; se revisa cada commit. | David | 2026-10-09 |

## 8. Verificación

Cada control implementado debe tener **cómo se comprueba**:

| Control | Cómo se verifica | Evidencia |
|---|---|---|
| Autorización por `user_id` | Test que intenta acceder a una tarea de otro usuario (IDOR) | `core/tests/test_authz.py` |
| Sin secretos en repo | `gitleaks detect --source .` (o barrido manual) | salida del comando |
| Postgres sin puerto público | `ss -tlnp` en el host no muestra 5432 | salida del comando |
| Contenedores endurecidos | `docker inspect agenda-core` muestra `no-new-privileges` y `User` no vacío | salida del comando |
| Idempotencia de envío | Test que reclama dos veces y solo envía una | `core/tests/test_worker_idempotency.py` |
| DST correcto | Test de cambio de hora con `zoneinfo` | `core/tests/test_timezone.py` |
| Salida del modelo validada | Test con JSON inválido → se rechaza y se pide reformular | `core/tests/test_nlp_guardrails.py` |
| Timeout del modelo | Test con servidor que no responde | `core/tests/test_nlp_timeout.py` |

## 9. Revisión

Este documento se revisa ante cada cambio relevante: nuevo endpoint, nuevo dato
sensible, nuevo proveedor externo, o incidente de seguridad.

| Fecha | Motivo | Autor |
|---|---|---|
| 2026-10-09 | Versión inicial (diseño) | Agente de diseño |
