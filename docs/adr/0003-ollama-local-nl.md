# ADR-003: Ollama local (`qwen3:8b`) para el lenguaje natural

- **Estado:** Aceptada
- **Fecha:** 2026-10-09
- **Decide:** David

---

## Contexto

Se quiere aceptar mensajes en lenguaje natural ("recuérdame llamar al dentista el
martes a las 5"), además de comandos. Eso requiere interpretar texto libre y
convertirlo en datos estructurados. El servidor ya ejecuta Ollama con los modelos
`qwen3:8b` y `llama3.1:8b` disponibles.

Restricciones que aplican:
- Privacidad: la agenda es personal; no conviene mandar su contenido a terceros.
- Coste: el proyecto es personal; no hay presupuesto para una API de pago.
- El servidor es de 8 núcleos y 13 GiB de RAM.

## Decisión

Usar el `qwen3:8b` local de Ollama como extractor de intenciones, con salida JSON
validada por el código y confirmación del usuario antes de actuar.

## Consecuencias

### Positivas
- Privacidad total: los datos no salen del servidor.
- Coste cero y sin claves de API.
- Sin dependencia de la disponibilidad de un proveedor externo.

### Negativas

> **Obligatoria.**

- **Menor precisión** que un modelo grande: habrá frases que necesiten
  confirmación o reformulación.
- **Consumo de recursos del host:** `qwen3:8b` ocupa RAM/CPU que comparte con
  otros servicios (Caddy, n8n, *arr, etc.). Hay que acotar `num_predict` y evitar
  llamadas innecesarias.
- **Latencia** superior a una API remota en algunos casos; se mitiga con timeout
  y con la primera capa determinista.
- Depende de que Ollama esté levantado y del modelo descargado.

### Neutras
- El prompt y el esquema de salida son los mismos que se usarían con otro modelo.

## Alternativas consideradas

### Alternativa A: API externa (p. ej. OpenAI/Anthropic)

- **Descripción:** llamar a un modelo remoto de alta calidad.
- **Pros:** mejor precisión y mantenimiento nulo.
- **Contras:** coste por uso; los datos de agenda salen del servidor.
- **Razón de rechazo:** privacidad y coste.

### Alternativa B: Solo reglas deterministas (sin IA)

- **Descripción:** parsear con expresiones y una gramática fija.
- **Pros:** determinista, rápido, sin recursos.
- **Contras:** no cubre la variabilidad real del lenguaje.
- **Razón de rechazo:** se usa **como primera capa**, pero por sí sola no basta.

## Validación

| Criterio | Objetivo | Se revisa el |
|---|---|---|
| Precisión del extractor | ≥ 90% en el dataset de `evals/` | Al cerrar la Fase 4 |
| El host no se degrada | El resto de servicios sigue respondiendo | Al cerrar la Fase 4 |

## Referencias

- `docs/AI.md`
- `docs/INVESTIGACION.md` §1
