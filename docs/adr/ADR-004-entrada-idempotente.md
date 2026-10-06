# ADR-004 — Entrada idempotente y "al menos una vez"

**Estado:** aceptada · **Relacionado:** `design.md` §0, §2.2, §17, §18

## Contexto

Meta y Telegram reintentan los webhooks si no reciben 200 a tiempo, y a veces los repiten igualmente. Los procesos se reinician a mitad de un trabajo. No se puede perder un mensaje ni duplicar una cita o un envío.

## Decisión

- **Guardar primero, contestar 200, procesar después**: el webhook verifica, inserta `inbound_event` con `UNIQUE(tenant_id, source, provider_event_id)` y encola en la misma transacción.
- Procesamiento con garantía **"al menos una vez"** + **idempotencia** en todo efecto externo: clave estable por evento (`ctx.idem`), tabla `side_effect` con `started` guardado y confirmado **antes** de llamar al sistema externo, y `done` en otra transacción.
- **Transacciones cortas**: nunca una transacción abierta durante llamadas externas.
- Envíos con estados explícitos; un envío ambiguo (timeout) pasa a `unknown` y **no se reintenta a ciegas**.
- Candado por conversación (hash sin teléfono) para todo escritor de la conversación + `conversation.version`, y advisory lock por calendario y día (con id de evento determinista) para evitar dobles reservas.
- Lo interno también entra por `inbound_event` (`source=system` para periódicas y tareas); los avisos de Google Calendar solo encolan una sincronización, que crea un evento por cambio.

## Alternativas consideradas

- **Procesar dentro del webhook**: riesgo de timeouts y reintentos del proveedor; se pierde trabajo si el proceso cae.
- **"Exactamente una vez"**: imposible de garantizar con sistemas externos; daría una falsa sensación de seguridad.

## Consecuencias

- Todo adaptador que escribe debe ser idempotente o consultable (p. ej. id de evento determinista en Calendar).
- Más estado que gestionar (`side_effect`, `send_intent`, estados `unknown`) y una pantalla de trabajos fallidos.

## Cuándo revisarla

- Se cambia la cola a SQS u otro sistema externo a la BD → patrón *outbox* para mantener la atomicidad.
