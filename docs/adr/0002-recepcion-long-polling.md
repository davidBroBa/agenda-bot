# ADR-002: Recepción de Telegram por long polling con gateway propio

- **Estado:** Aceptada
- **Fecha:** 2026-10-09
- **Decide:** David

---

## Contexto

El bot necesita recibir los mensajes de Telegram. Las dos formas de recibirlos
son *webhook* (Telegram hace una petición entrante a una URL nuestra) o *long
polling* (nosotros preguntamos a Telegram). El servidor está detrás de NAT, no
tenemos `sudo`, y los puertos 80/443 los gestiona Caddy para **otros** dominios.

Restricciones que aplican:
- Sin `sudo` (no hay `NOPASSWD`).
- No se quieren introducir cambios en Caddy ni en `cloudflared`.
- El sistema no debe requerir abrir un puerto nuevo.

## Decisión

Recibir los mensajes con un componente propio (`agenda-gateway`) que hace *long
polling* contra `getUpdates` de Telegram y reenvía internamente a n8n.

## Consecuencias

### Positivas
- No hace falta abrir ningún puerto entrante ni tocar DNS/TLS.
- Funciona tras NAT sin configuración de red.
- Se controla el *offset* y la deduplicación de `update_id`.

### Negativas

> **Obligatoria.**

- **Latencia y consumo:** el polling mantiene conexiones largas y puede llegar a
  límites de la API si se abusa; hay que regular el *timeout* de `getUpdates`.
- **Escalado:** no se pueden correr **dos** gateways a la vez (Telegram asigna
  cada update a un solo consumidor por *offset*); es un punto único. Aceptable
  para uso personal, bloqueante para multiusuario.
- Requiere un proceso siempre vivo; si se cae, los mensajes quedan en la cola de
  Telegram y se pierden si pasan demasiado tiempo.

### Neutras
- Telegram ofrece webhook y polling con la misma API de envío (`sendMessage`).

## Alternativas consideradas

### Alternativa A: Webhook a través de Caddy + subdominio

- **Descripción:** exponer un endpoint en un subdominio con certificado de Caddy.
- **Pros:** menor latencia; sin proceso de polling; escala mejor.
- **Contras:** requiere editar el `Caddyfile` (**necesita `sudo`**), un registro
  DNS y exponer un puerto a Internet; superficie de ataque mayor.
- **Razón de rechazo:** exige `sudo` y cambios en servicios ajenos al proyecto.

### Alternativa B: Túnel `cloudflared`

- **Descripción:** exponer el webhook por el túnel de Cloudflare ya presente.
- **Pros:** no requiere abrir puertos.
- **Contras:** depende de un servicio y una configuración que no gestionamos;
  el túnel existente es ad-hoc y su configuración no es nuestra.
- **Razón de rechazo:** acoplamiento a infraestructura ajena.

## Validación

| Criterio | Objetivo | Se revisa el |
|---|---|---|
| No abre puertos | `ss -tlnp` del host no muestra puertos nuevos del proyecto | Al cerrar la Fase 1 |
| Latencia de recepción | < 5 s percibidos | Al cerrar la Fase 1 |

## Referencias

- `docs/NETWORK.md` §1-3
- `docs/ARQUITECTURA.md` §2
