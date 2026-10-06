# ADR-002 — Multitenancy con Row-Level Security

**Estado:** aceptada · **Relacionado:** `design.md` §0, §17, §23.1, §23.2

## Contexto

Muchos negocios pequeños comparten la plataforma. Un error de código que muestre datos de un negocio a otro es inaceptable, y operar una base de datos por cliente no es sostenible para una persona.

## Decisión

- **Una base de datos PostgreSQL 18 compartida**; toda tabla con datos de cliente lleva `tenant_id`.
- **RLS forzada** (`ENABLE` + `FORCE`) con la política `tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid` (con conexiones persistentes el valor vacío es `''`, no NULL).
- El tenant se fija **solo** con `tenant_tx(tenant_id)`: transacción corta + `SET LOCAL app.tenant_id`, nunca anidada. Sin tenant fijado no se ve ninguna fila.
- Roles: `app_owner` (migraciones), `app_runtime` (web y worker, sin `BYPASSRLS`), `app_operator` (consola y admin, con `BYPASSRLS`).
- Claves foráneas compuestas `(tenant_id, x_id)`; entradas HTTP y trabajos que recorren tenants resueltos con funciones `SECURITY DEFINER` mínimas, que devuelven solo ids, estado y lo mínimo para verificar la firma (con `SET search_path = pg_catalog, public`).
- Test genérico de aislamiento que recorre todas las tablas con `tenant_id` y bloquea el merge.

## Alternativas consideradas

- **Base de datos o esquema por cliente**: aislamiento fuerte, pero migraciones, conexiones y backups multiplicados.
- **Filtrar solo en el ORM**: un olvido en una consulta expone datos; la BD no lo impediría.

## Consecuencias

- El aislamiento no depende de que el código acierte siempre: falla de forma segura.
- Toda migración de tabla nueva debe crear su política (lo vigila el test); las que tocan RLS o roles requieren revisión humana.
- Las tablas de Procrastinate no tienen RLS: los trabajos solo llevan ids y los candados no llevan datos personales.

## Cuándo revisarla

- Un cliente exige, por obligación legal o contractual, una base de datos propia.
- El volumen de un tenant afecta al resto de forma que no se resuelve con límites de concurrencia.
