# ADR-001: Reutilizar el n8n existente en lugar de desplegar uno dedicado

- **Estado:** Aceptada
- **Fecha:** 2026-10-09
- **Decide:** David

---

## Contexto

El servidor `srv` ya tiene un contenedor `n8n` (`n8nio/n8n:latest`, v2.41.6)
escuchando en `0.0.0.0:5678`, arrancado a mano y con `restart=no`. Corre como
`root` con `HOME=/root`, así que sus datos reales están en
`/root/.n8n/database.sqlite` **dentro de la capa del contenedor**; el bind
`/DATA/AppData/n8n -> /home/node/.n8n` está vacío y no se usa. **No hay
PostgreSQL** en el servidor.

Restricciones que aplican:
- El usuario pidió explícitamente **reutilizar** el n8n existente.
- El encargo prohíbe eliminar o reemplazar servicios previos.
- No hay `sudo` disponible.

## Decisión

Usar el contenedor `n8n` existente como motor de orquestación, añadiendo nuestros
workflows como JSON versionados en el repositorio (que pasa a ser la fuente de
verdad).

## Consecuencias

### Positivas
- Un contenedor menos que mantener y un stack más ligero.
- Aprovecha que n8n ya está funcionando y accesible internamente (`172.17.0.2:5678`).

### Negativas

> **Obligatoria.**

- **Riesgo de pérdida de datos:** los workflows y credenciales viven en la capa
  del contenedor. Si alguien recrea el contenedor, se pierden. Se mitiga
  versionando los workflows en el repo, pero las **credenciales** no se versionan.
- `restart=no`: tras un reinicio del servidor, n8n no vuelve solo. El sistema
  depende de que alguien (o un `restart` que no controlamos) lo levante.
- Acoplamiento a la versión `latest`: una actualización puede romper los workflows.

### Neutras
- La latencia interna entre nuestros contenedores y n8n es despreciable.

## Alternativas consideradas

### Alternativa A: n8n dedicado (contenedor propio + volumen persistente)

- **Descripción:** desplegar un segundo n8n en el Compose del proyecto, con
  volumen nombrado y `restart: unless-stopped`.
- **Pros:** aislamiento total; datos persistentes; política de reinicio propia.
- **Contras:** un contenedor y una base más que mantener; dos interfaces de n8n.
- **Razón de rechazo:** el usuario pidió reutilizar el existente y el encargo
  prohíbe añadir servicios sin necesidad.

### Alternativa B: prescindir de n8n (solo Python)

- **Descripción:** que el gateway decida y llame directamente al core, sin n8n.
- **Pros:** menos dependencias; más simple y testeable.
- **Contras:** se pierde la orquestación visual y la capacidad de añadir pasos sin
  tocar código, que fue un objetivo del proyecto.
- **Razón de rechazo:** el usuario eligió n8n como pieza central.

## Validación

| Criterio | Objetivo | Se revisa el |
|---|---|---|
| Los workflows son reimportables | `docker cp` del JSON + import reproduce el flujo | Al cerrar la Fase 4 |
| El sistema no depende de datos efímeros | El repo contiene todos los workflows | Al cerrar la Fase 4 |

## Referencias

- `docs/INVESTIGACION.md` §1-2
- `docs/ARQUITECTURA.md` §5
