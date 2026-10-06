# ADR-003 — Packs y contrato PackContext

**Estado:** aceptada · **Relacionado:** `design.md` §0, §4.2, §19

## Contexto

Cada tipo de negocio (peluquería, autobuses, gimnasio…) tiene su lógica. Queremos añadir negocios y tipos de negocio sin tocar la plataforma ni refactorizar, y sin inventar un lenguaje de flujos difícil de mantener.

## Decisión

- Cada tipo de negocio es un **pack** en `packs/<nombre>/`, descubierto automáticamente por el registro.
- Un pack declara: esquema de config (pydantic), **recorridos en Python** (`Journeys` de bloques), handlers por disparador (`message`, `payload:<prefijo>`, `form`, `calendar`, `task:<nombre>`, periódicas; [FUTURO] `webhook:<nombre>`), todos con la firma `handler(event, ctx) -> HandleResult`, conectores, datasets y plantillas.
- El pack recibe **solo un `PackContext`** (config, conectores, knowledge, datasets, reloj, idempotencia…) y devuelve un `HandleResult` (respuestas, acciones, estado). **Nunca toca la BD ni las APIs directamente.**
- Cada negocio concreto varía por **config** (y datasets), sin código por tenant.
- Única excepción a "sin Django en los packs": `packs/*/storage.py`, detrás de un `Protocol` expuesto en `ctx.store`.

## Alternativas consideradas

- **Recetas YAML / DSL por tenant**: flexibles, pero tienden a convertirse en un lenguaje de programación sin herramientas. Quedan como [FUTURO], solo si la regla de los tres lo justifica.
- **Packs con acceso directo al ORM**: más rápido al principio; acopla los packs a la BD y obliga a tests con Postgres.

## Consecuencias

- Los packs se prueban sin BD con escenarios y conectores mock.
- Contrato pequeño y estable: cambiarlo afecta a todos los packs, así que se cambia poco y con cuidado.
- Variaciones entre tenants de un mismo pack se resuelven con opciones de config; si no bastan, aparece la necesidad de recetas.

## Cuándo revisarla

- La misma variación aparece en tres tenants y la config no la cubre → recetas o variantes por tenant.
- Un pack necesita algo que `PackContext` no ofrece de forma repetida → se amplía el contrato, no se salta.
