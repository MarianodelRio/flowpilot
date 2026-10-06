# ADR-001 — Monolito modular Django + Procrastinate

**Estado:** aceptada · **Relacionado:** `design.md` §0, §4, §18.2, §14

## Contexto

Una sola persona construye y opera la plataforma. Hay que atender webhooks rápido, procesar trabajo en segundo plano (conversaciones, envíos, barridos, sincronizaciones) y añadir negocios nuevos sin reescribir nada. La carga prevista es pequeña (~50 negocios, picos de 1–2 mensajes/s).

## Decisión

- **Un solo proyecto Python 3.14 + Django 5.2 LTS**, dividido en módulos (`core/` sin Django, `flowpilot/` con Django, `packs/`), con la regla de dependencias comprobada por `import-linter`.
- **Una imagen, dos procesos**: `web` (gunicorn `gthread`; verifica, guarda y encola) y `worker` (Procrastinate).
- **Cola en el propio PostgreSQL con Procrastinate 3.x**: 4 colas (`inbound`, `outbound`, `scheduled`, `heavy`), candados por conversación, tareas periódicas integradas (sin proceso scheduler aparte).

## Alternativas consideradas

- **Microservicios**: más despliegues, más red, más fallos parciales; sin beneficio a esta escala.
- **Celery + Redis / SQS**: una pieza más que operar; Postgres ya da transacciones compartidas con los datos (el trabajo solo existe si hay COMMIT).
- **FastAPI**: menos piezas incluidas (admin, auth, migraciones); habría que montarlas.

## Consecuencias

- Un solo despliegue, un solo sitio donde mirar; los tests de `core` y packs corren sin BD.
- Sin RDS Proxy (Procrastinate usa `LISTEN/NOTIFY`): hay que vigilar el número de conexiones por tarea.
- La cola comparte recursos con los datos: una saturación de la BD afecta a ambas.

## Cuándo revisarla

- Postgres como cola llega a su límite (miles de trabajos/s) → cola en SQS detrás de la misma interfaz, con patrón *outbox*.
- Un módulo necesita escalar o desplegarse de forma independiente y separar servicios `worker` no basta.
- Django 6.2 LTS (abril de 2027): planificar la migración antes del fin de soporte de 5.2 (abril de 2028).
