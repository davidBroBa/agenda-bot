# Topología de Red — Agenda inteligente y recordatorios por Telegram

> Documenta **cómo viaja el tráfico**. Si algo no está aquí, no debería existir.

---

## 1. Diagrama

```
[ Usuario / movil ]
   |
   |  (el usuario habla con Telegram, no con nosotros)
   v
[ Telegram Cloud ]
   ^
   |  (1) HTTPS saliente: getUpdates (long polling)  y  sendMessage
   |
[ Host srv (Ubuntu) ]
   |
   +-- agenda-gateway --------+
   |       |                  |
   |       | (2) HTTP interno  | (3) HTTPS a Telegram
   |       v                  |
   |   [ n8n :5678 ]          |
   |       | (4) HTTP interno  |
   |       v                  |
   |   [ agenda-core :8080 ]  |
   |       |                  |
   |       +--> [ postgres:17 :5432 ]   red interna, NO publica
   |       +--> [ ollama :11434 ]       (host, existente)
   |
   +-- agenda-worker ---------> Telegram (sendMessage) + postgres
```

**Clave:** **no hay tráfico entrante** a nuestros servicios. El bot obtiene los
mensajes por *long polling* (nosotros salimos). Esto evita abrir puertos, tocar
DNS y usar `sudo`.

## 2. Inventario de servicios

| Servicio | Host | Puerto | Expuesto | Autenticado | Plan de respaldo |
|---|---|---|---|---|---|
| agenda-gateway | red Docker `agenda` | — (sin puerto) | No | Lista blanca + secreto | Se reinicia solo (`restart: unless-stopped`) |
| agenda-core | red Docker `agenda` | 8080 | No (interno) | Red interna | Se reinicia solo |
| agenda-worker | red Docker `agenda` | — | No | — | Se reinicia solo |
| postgres:17 | red Docker `agenda` | 5432 | **No** | Usuario/contraseña | `pg_dump` periódico |
| n8n (existente) | host | 5678 | **Sí (0.0.0.0)** | Login de n8n | **No recrear**; workflows versionados |
| ollama (existente) | host | 11434 | **Sí (0.0.0.0)** | No | Fuera de nuestro control |

## 3. Puertos y exposición

| Puerto | Servicio | Público | Firewall | Justificación |
|---|---|---|---|---|
| 22 | SSH | (según host) | — | Solo por clave; no lo gestionamos |
| 80 | HTTP (Caddy) | Sí | — | Otros dominios; solo redirección y ACME |
| 443 | HTTPS (Caddy) | Sí | — | `davidamador.dev` de otros proyectos |
| 5432 | PostgreSQL | **No** | Red Docker | Solo red interna |
| 8080 | agenda-core | **No** | Red Docker | Solo n8n/worker |
| 5678 | n8n | Sí (preexistente) | — | Fuera del encargo; se recomienda ligarlo a `127.0.0.1` |
| 11434 | Ollama | Sí (preexistente) | — | Fuera del encargo; se recomienda ligarlo a `127.0.0.1` |

**Regla:** la base de datos nunca se expone a Internet. Nunca.

## 4. DNS

| Registro | Tipo | Valor | TTL | Propósito |
|---|---|---|---|---|
| — | — | — | — | **No se añade ningún registro DNS.** El bot no necesita nombre ni certificado propios |

Verificación: no aplica.

## 5. TLS

| Aspecto | Valor |
|---|---|
| Versión mínima | TLS 1.2 (a los servicios existentes) |
| Versión preferida | TLS 1.3 |
| Cifrado hacia Telegram | HTTPS gestionado por la API de Telegram |
| Certificado | No emitimos certificado propio |
| Renovación automática | No aplica |
| HSTS | No aplica (no servimos web propia) |
| Verificación | `curl -sI https://api.telegram.org` responde |

## 6. Reglas de tráfico

| Origen | Destino | Puerto | Protocolo | Regla |
|---|---|---|---|---|
| Internet | nuestros servicios | — | — | **Bloqueado** (no hay ingresos) |
| agenda-gateway | api.telegram.org | 443 | HTTPS | Permitido (salida) |
| agenda-core/worker | api.telegram.org | 443 | HTTPS | Permitido (salida) |
| agenda-gateway | n8n (host) | 5678 | HTTP | Permitido (interno) |
| n8n | agenda-core | 8080 | HTTP | Permitido (interno) |
| agenda-core/worker | postgres | 5432 | TCP | Permitido (interno) |
| agenda-core | ollama (host) | 11434 | HTTP | Permitido (interno) |
| Internet | postgres | 5432 | — | **Bloqueado** |

## 7. Política de caché

| Recurso | `Cache-Control` |
|---|---|
| Respuestas HTTP del core | `no-store` (datos personales) |
| Respuestas de error | `no-store` |
| Caché del parser NL | local por hash de consulta |

## 8. Timeouts y reintentos

| Dependencia | Timeout | Reintentos | Notas |
|---|---|---|---|
| PostgreSQL | 5 s | Sin reintento en escrituras | Conexión con pool |
| API de Telegram | 10 s | Sí (idempotente por `idempotency_key`) | Backoff exponencial + jitter |
| n8n (gateway →) | 10 s | Sí | El gateway reintenta el envío |
| Ollama | 20 s | 1 reintento | Modelo local; si falla, se pide reformular |

## 9. CI/CD

```
push -> lint + typecheck -> tests -> (gates locales) -> (deploy manual en el servidor)
```

| Etapa | Trigger | Comando | Bloquea deploy |
|---|---|---|---|
| Lint | cada push | `ruff check .` | Sí |
| Typecheck | cada push | `mypy --strict src` | Sí |
| Tests | cada push | `pytest` | Sí |
| Auditoría | cada push | `pip-audit` | Sí |
| Build | main | `docker compose build` | Sí |
| Deploy | manual, en el servidor | `docker compose up -d` | — |

> Los gates se ejecutan en local sobre el código descargado (el servidor no tiene
> `pwsh`). Se documentarán en `docs/GATES.md`.

## 10. Monitoreo de red

| Métrica | Umbral de alerta | Herramienta |
|---|---|---|
| Latencia de `getUpdates` | > 5 s | log del gateway |
| Tasa de error de envío a Telegram | > 1% | log del worker |
| Timeouts a Ollama | > 10% | log del core |
| Recordatorios atrasados | > 5 min | healthcheck |

## 11. Verificación

```bash
# Salida a Telegram
curl -sS -m10 -o /dev/null -w "%{http_code}\n" https://api.telegram.org

# Postgres NO escucha en el host
ss -tlnp | grep 5432   # no debe aparecer

# Servicios de nuestro proyecto
docker compose ps
```

- [ ] Base de datos sin exposición pública
- [ ] Ningún puerto de nuestro stack publicado en el host
- [ ] Timeout en toda llamada externa
- [ ] Reintentos solo en operaciones idempotentes
- [ ] (Recomendado a David) n8n y Ollama ligados a `127.0.0.1`
