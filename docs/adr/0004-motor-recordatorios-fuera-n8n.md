# ADR-004: Motor de recordatorios fuera de n8n

- **Estado:** Aceptada
- **Fecha:** 2026-10-09
- **Decide:** David

---

## Contexto

El sistema debe emitir recordatorios puntuales y recuperarlos si el servicio
estuvo caído. n8n podría hacerlo con un workflow programado, pero el n8n que se
va a reutilizar (ADR-001) tiene dos problemas conocidos: `restart=no` y datos en
la capa del contenedor. La puntualidad de los avisos es la **razón de ser** del
proyecto.

Restricciones que aplican:
- Un recordatorio que no llega es el peor fallo posible del sistema.
- No se puede garantizar que el n8n existente esté siempre arriba.

## Decisión

Implementar el motor de recordatorios como un proceso propio (`agenda-worker`,
parte del paquete `agenda`) que cada 30 s reclama los avisos vencidos con
`SELECT ... FOR UPDATE SKIP LOCKED`, envía, registra y recupera atrasados.

## Consecuencias

### Positivas
- La ruta crítica **no depende de n8n**.
- Control transaccional fino: idempotencia con `idempotency_key` y `dedupe_key`.
- Recuperación tras caída dentro de la ventana de la política.
- Testeable de forma aislada y determinista.

### Negativas

> **Obligatoria.**

- **Más código que mantener** (bucle, reaper, envío, errores) frente a un workflow.
- Un servicio más en el Compose.
- Hay que cuidar la concurrencia y los recordatorios atrasados (si el sistema
  estuvo caído, decidir si se avisa tarde o se descarta).

### Neutras
- n8n conserva un disparo de respaldo (`04-dispatch` → `POST /tick`) por si el
  worker no está corriendo.

## Alternativas consideradas

### Alternativa A: Todo en n8n (Schedule Trigger + Query + Telegram)

- **Descripción:** un workflow programado que consulta y envía.
- **Pros:** sin código propio; visual.
- **Contras:** depende del n8n frágil; garantías de idempotencia y recuperación
  más difíciles de expresar y de probar.
- **Razón de rechazo:** poner la ruta crítica en el componente menos fiable.

### Alternativa B: Colas y planificador externo (p. ej. Redis + worker)

- **Descripción:** usar un broker de colas con programación.
- **Pros:** patrón robusto y escalable.
- **Contras:** una dependencia más (Redis) para un uso personal.
- **Razón de rechazo:** complejidad innecesaria; PostgreSQL ya cubre el reclamo
  transaccional con `FOR UPDATE SKIP LOCKED`.

## Validación

| Criterio | Objetivo | Se revisa el |
|---|---|---|
| Sin duplicados | Un recordatorio genera un solo envío | Al cerrar la Fase 3 |
| Recuperación | Tras una caída de 10 min, los avisos atrasados se emiten una vez | Al cerrar la Fase 3 |
| Independencia de n8n | Apagar n8n no detiene los recordatorios | Al cerrar la Fase 3 |

## Referencias

- `docs/ARQUITECTURA.md` §2, §4
- ADR-001
