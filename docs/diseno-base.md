# design.md — Diseño técnico de la plataforma

> **Qué es este documento.** El diseño completo de la parte técnica: qué piezas hay, cómo se hablan, dónde vive cada cosa, cómo se programa un negocio nuevo, cómo se prueba, cómo se despliega y en qué infraestructura corre. El objetivo es que, con este documento delante, **solo queden dudas de implementación** (cómo escribir tal función), no de diseño (dónde va, quién la llama, cómo se despliega).
>
> **Para quién.** Para ti (Mariano) y para los agentes de programación. Cada sección empieza con un bloque **"En pocas palabras"** en lenguaje llano y después entra en el detalle técnico. Los términos técnicos están en el [glosario](#24-glosario) del final.
>
> **Relación con otros documentos.**
> - `NEGOCIO.md`: el porqué (mercado, precios, legal). Sus secciones técnicas (§6–13) quedan **sustituidas por este documento**.
> - `docs/adr/`: una página por decisión importante (las ADR enlazadas aquí).
> - `docs/migracion.md` (pendiente): cómo pasar del código actual del repo a este diseño.
> - `AGENTS.md`: las reglas de este documento resumidas para los agentes.
>
> **Convenciones.** **[DECIDIDO]**: no se discute salvo evidencia nueva. **[VERIFICAR]**: dato externo que hay que confirmar antes de usarlo. **[FUTURO]**: está diseñado, pero no se construye todavía.

---

## Índice

0. [Resumen de decisiones](#0-resumen-de-decisiones)
1. [Visión general](#1-visión-general)
2. [Principios y no-objetivos](#2-principios-y-no-objetivos)
3. [Conceptos clave](#3-conceptos-clave)
4. [Arquitectura de la aplicación](#4-arquitectura-de-la-aplicación)
5. [Packs: cómo conviven los distintos negocios](#5-packs-cómo-conviven-los-distintos-negocios)
6. [Canales](#6-canales)
7. [Conectores](#7-conectores)
8. [Inteligencia artificial](#8-inteligencia-artificial)
9. [Datos](#9-datos)
10. [Fiabilidad: que no se pierda ni se duplique nada](#10-fiabilidad-que-no-se-pierda-ni-se-duplique-nada)
11. [Seguridad y secretos](#11-seguridad-y-secretos)
12. [Configuración: qué vive dónde](#12-configuración-qué-vive-dónde)
13. [Consola y Django Admin](#13-consola-y-django-admin)
14. [Entornos y desarrollo local](#14-entornos-y-desarrollo-local)
15. [Calidad y revisión](#15-calidad-y-revisión)
16. [Construcción y despliegue](#16-construcción-y-despliegue)
17. [Infraestructura en AWS](#17-infraestructura-en-aws)
18. [Observabilidad y alertas](#18-observabilidad-y-alertas)
19. [Operación técnica](#19-operación-técnica)
20. [Escalado](#20-escalado)
21. [Estructura del repositorio](#21-estructura-del-repositorio)
22. [Migración desde el código actual](#22-migración-desde-el-código-actual)
23. [Pendiente de verificar y decisiones abiertas](#23-pendiente-de-verificar-y-decisiones-abiertas)
24. [Glosario](#24-glosario)

---

## Architecture

### 0. Resumen de decisiones

| # | Tema | Decisión | ADR |
|---|---|---|---|
| 1 | Lenguaje y framework | **Python + Django 5.2 LTS** (soporte hasta abril de 2028) | ADR-001 |
| 2 | Forma de la aplicación | **Un solo proyecto (monolito modular)** con dos procesos: `web` y `worker`, misma imagen | ADR-002 |
| 3 | Clientes (multitenant) | **Una base de datos PostgreSQL compartida**, cada fila con `tenant_id`, aislamiento con **Row-Level Security** | ADR-003 |
| 4 | Trabajo en segundo plano | **Procrastinate** (cola guardada en el propio Postgres) con 4 colas: `inbound`, `outbound`, `scheduled`, `heavy` | ADR-004 |
| 5 | Entrada de mensajes | **Guardar primero, contestar 200, procesar después** (nada se pierde) | ADR-005 |
| 6 | Lógica de negocio | **Packs en Python** construidos con **bloques reutilizables**; cada cliente varía por **config → receta → variante en código**. El intérprete YAML actual se retira | ADR-006 |
| 7 | Dónde vive cada cosa | **Híbrido**: lógica y recetas en el repositorio (con revisión y tests); datos de negocio (precios, horarios, textos, knowledge) en la base de datos (cambio inmediato) | ADR-007 |
| 8 | Conectores | Servicios compartidos **por categoría con métodos tipados**; reintentos, timeouts y circuit breaker centralizados | ADR-008 |
| 9 | IA | El LLM **solo genera o clasifica texto** acotado al negocio; nunca decide reservas ni cambia el flujo por su cuenta | ADR-009 |
| 10 | WhatsApp | El cliente es dueño de su número y su cuenta, y paga a Meta. Dos modos: app del cliente (sin alta) y app de la plataforma | ADR-010 |
| 11 | Interfaz interna | **Consola** (Django + HTMX) para operar + **Django Admin** para datos en bruto. Portal de cliente [FUTURO] | ADR-011 |
| 12 | Consola del dueño | **Bot de Telegram del dueño** (avisos, resumen, comandos) | ADR-012 |
| 13 | Infraestructura | **AWS: ECS Fargate (ARM) + RDS PostgreSQL + ALB + S3 + KMS**, región **eu-south-2 (España)** [VERIFICAR servicios], plan B eu-west-1 | ADR-013 |
| 14 | Infraestructura como código | **AWS CDK en Python** | ADR-014 |
| 15 | Cuentas de AWS | **Dos cuentas** (staging y prod) bajo AWS Organizations | ADR-015 |
| 16 | Despliegue | GitHub Actions con OIDC → imagen en ECR → tarea `migrate` → despliegue progresivo con vuelta atrás automática. **Staging automático, prod con aprobación manual** | ADR-016 |
| 17 | Secretos | Credenciales de clientes **cifradas con KMS** en la BD; secretos de plataforma en **SSM Parameter Store / Secrets Manager** | ADR-017 |
| 18 | Calidad | Puertas de CI obligatorias + **tests de escenarios por pack y por cliente** + revisión humana en todo PR | ADR-018 |

---

### 1. Visión general

> **En pocas palabras.** La plataforma es **un programa** que corre en la nube de Amazon. Recibe mensajes y avisos de muchos sitios (WhatsApp, Telegram, formularios, tareas programadas), averigua **de qué negocio son**, ejecuta **la lógica de ese tipo de negocio** (la del "pack": peluquería, autobuses, gimnasio…) con **la configuración de ese negocio concreto** (sus precios, horarios, textos) y responde, usando si hace falta **conectores** (Google Calendar, la IA, hojas de cálculo…). Todo queda registrado: qué entró, qué se hizo, qué salió y cuánto costó.

#### 1.1 Las piezas

```
                      ┌────────────────────────── AWS (cuenta prod) ──────────────────────────┐
  WhatsApp (Meta) ─┐  │                                                                        │
  Telegram ────────┤  │   ALB (puerta de entrada HTTPS)                                        │
  Web chat ────────┼──┼──►  ├─ api.<dominio>      ─┐                                           │
  Formularios ─────┤  │     └─ consola.<dominio>  ─┤                                           │
  Webhooks ────────┘  │                            ▼                                           │
                      │                    ┌──────────────┐        ┌────────────────────────┐  │
                      │                    │  web (Django)│───────►│ PostgreSQL (RDS)       │  │
                      │                    │  N copias    │        │  · datos de todo       │  │
                      │                    └──────────────┘        │  · cola de trabajos    │  │
                      │                                            └───────────┬────────────┘  │
                      │                    ┌──────────────┐                    │               │
                      │                    │ worker       │◄───────────────────┘               │
                      │                    │ N copias     │── conectores ──► Google, IA, Meta… │
                      │                    └──────────────┘                                    │
                      │   S3 (ficheros) · KMS (claves) · CloudWatch (logs, métricas, alarmas)  │
                      └────────────────────────────────────────────────────────────────────────┘
```

| Pieza | Qué es | Qué hace |
|---|---|---|
| **web** | Django ejecutándose con gunicorn | Recibe webhooks, sirve la consola y el admin, el web chat y `/health`. **No procesa mensajes**: los guarda y los encola |
| **worker** | El mismo código, ejecutando el proceso de Procrastinate | Procesa la cola: ejecuta la lógica de los packs, llama a conectores, envía mensajes, dispara recordatorios |
| **PostgreSQL** | Base de datos gestionada por AWS (RDS) | Guarda todo, incluida la cola de trabajos |
| **ALB** | Balanceador de carga de AWS | Puerta de entrada HTTPS; reparte peticiones entre las copias de `web` |
| **S3** | Almacenamiento de ficheros | Documentos y audios recibidos, exportaciones |
| **KMS** | Servicio de claves | Cifra las credenciales de los clientes |
| **CloudWatch** | Logs y métricas de AWS | Registros, gráficas, alarmas |

#### 1.2 La vida de un mensaje (de punta a punta)

Ejemplo: una clienta escribe "quiero cita" al WhatsApp de la Peluquería Sur.

1. **Meta** envía un aviso (webhook) a `https://api.<dominio>/webhooks/whatsapp/<id-del-canal>`.
2. **web** comprueba la firma (que de verdad viene de Meta), identifica el canal → el negocio, **guarda el evento** en la tabla `inbound_event` (si ya existía porque Meta lo reenvió, no hace nada más), **encola** un trabajo `process_event` y contesta **200 OK**. Tarda menos de 100 ms.
3. **worker** coge el trabajo. Hay un candado por conversación: si la misma clienta manda dos mensajes seguidos, se procesan en orden, nunca a la vez.
4. **worker** carga la conversación (en qué paso está), la config de la peluquería y su receta, y pasa el mensaje al **pack de peluquería**.
5. El pack ejecuta el bloque actual (p. ej. "elegir servicio") y devuelve **respuestas** ("¿Qué servicio quieres?" + lista) y **acciones** (guardar el estado, programar un recordatorio…). Si necesita datos externos, llama a un **conector** (Google Calendar para ver huecos).
6. **worker** guarda en una sola transacción el nuevo estado, las respuestas como "intenciones de envío" (`send_intent`) y las tareas programadas, y encola los envíos.
7. **worker** (cola `outbound`) envía cada mensaje a Meta, guarda el id que devuelve Meta y marca el envío como hecho.
8. Más tarde llegan de Meta los avisos de "entregado" y "leído", con la categoría de precio: se actualiza el mensaje y su coste.

Si algo falla a mitad (se reinicia un worker, Google no responde), el trabajo se reintenta y, gracias a la idempotencia (§10), **no se duplica nada**.

---

### 2. Principios y no-objetivos

> **En pocas palabras.** Reglas que guían todas las decisiones, y lista de cosas que **no** vamos a construir para no complicarnos.

#### 2.1 Principios

1. **Simple antes que flexible.** Una base de datos, un proyecto, una imagen. Se añade complejidad solo con un problema real delante.
2. **Lo genérico en la plataforma, lo específico en los packs.** La plataforma no sabe qué es una peluquería.
3. **Lógica en código, datos en configuración.** Programar en Python; los precios, horarios y textos se editan sin programar.
4. **Nada se pierde y nada se duplica.** Todo evento se guarda antes de procesarse y todo efecto externo es idempotente.
5. **Determinista por defecto, IA solo donde aporta.** Horarios, disponibilidad y precios salen de datos, nunca de la IA.
6. **Todo cambio de lógica pasa por tests y revisión.** Los de datos se validan contra un esquema.
7. **Portable.** El código no depende de AWS directamente salvo detrás de interfaces: si mañana cambiamos de nube, se cambia infraestructura, no código.
8. **Medir desde el primer día**: mensajes, costes, tiempos y fallos por cliente.
9. **Diseñado para una persona**: lo que no se automatiza o no se puede operar fácilmente, no se hace.

#### 2.2 No-objetivos (no se construye)

Microservicios · Kubernetes · una base de datos o un contenedor por cliente · un lenguaje propio de flujos (DSL) · editor visual de flujos · inbox omnicanal propio (si hace falta: Chatwoot) · CRM propio · facturación propia (se usa software homologado) · app móvil · disponibilidad 24/7 garantizada · agentes de IA autónomos.

---

### 4. Arquitectura de la aplicación

> **En pocas palabras.** Es un solo proyecto Django dividido en "módulos" (carpetas con una responsabilidad cada una). Se ejecuta de dos maneras: como **web** (atiende peticiones) y como **worker** (hace el trabajo de fondo). Hay una regla de quién puede usar a quién, para que la plataforma nunca dependa de un negocio concreto.

#### 4.1 Procesos

| Proceso | Comando | Copias | Responsabilidad | Lo que NO hace |
|---|---|---|---|---|
| `web` | `gunicorn config.wsgi` | 1 → N (autoescala) | Webhooks (verificar, guardar, encolar), consola, admin, web chat, `/health` | Llamar a conectores, ejecutar packs, enviar mensajes |
| `worker` | `python manage.py procrastinate worker --queues=…` | 1 → N (autoescala) | Procesar eventos, enviar mensajes, tareas programadas y periódicas | Atender HTTP |
| `migrate` | `python manage.py migrate && python manage.py sync_packs` | Tarea puntual en cada despliegue | Actualizar el esquema de la BD y registrar las versiones de packs y recetas | — |

**[DECIDIDO]** No hay un proceso "scheduler" aparte: las tareas periódicas las lanza el propio worker (Procrastinate tiene tareas periódicas integradas y garantiza que solo una copia las lanza).

#### 4.2 Módulos (apps de Django) y regla de dependencias

```
src/
  core/            Python puro, SIN Django: tipos de dominio, contratos, motor de conversación, bloques base
  platform/        Django: tenancy, canales, conectores, datos, cola, runtime de eventos, consola
     tenancy/        Tenant, membresías, contexto de tenant, RLS
     channels/       whatsapp/, telegram/, webchat/, owner_bot/ (adaptadores + webhooks)
     connectors/     registro, políticas (reintentos, circuit breaker), adaptadores por categoría
     conversations/  Contact, Conversation, Message, Handoff
     events/         InboundEvent, runtime (evento → pack → acciones), SendIntent
     scheduling/     ScheduledTask, tareas periódicas
     credentials/    Credential (cifrado KMS)
     usage/          métricas de uso y coste, cuotas, informe mensual
     audit/          AuditEvent
     console/        Consola (vistas Django + HTMX)
  packs/           Los negocios: salon/, bus/, gym/ … (dependen de core y de la API de packs)
  config/          settings, urls, wsgi
```

**Regla de dependencias [DECIDIDO]** (la comprueba `import-linter` en CI):

```
packs/*   ──►  core   (y la "API de packs": core.packs, core.blocks, core.connectors.interfaces)
platform  ──►  core
platform  ──►  packs  SOLO a través del registro de packs (descubrimiento), nunca importando un pack concreto
core      ──►  nada del proyecto (ni Django, ni platform, ni packs)
packs/A   ──X  packs/B   (un pack no importa otro; lo compartido sube a core/blocks)
```

**Por qué importa:** cualquiera (persona o agente) puede trabajar en el pack de autobuses sin riesgo de romper la peluquería ni la plataforma, y los tests de `core` y de los packs corren sin base de datos.

#### 4.3 Runtime de eventos (el corazón)

```python
# platform/events/runtime.py  (pseudocódigo)
def process_event(event_id):
    event = InboundEvent.get(event_id)
    with tenant_context(event.tenant_id):                 # fija RLS (§9.2)
        domain_event = to_domain_event(event)             # InternalMessage / FormSubmitted / TaskDue / Webhook
        tenant = load_tenant(event.tenant_id)
        assignment = tenant.pack_assignment                # qué pack, qué receta, qué config
        pack = registry.get(assignment.pack_name)
        ctx = build_context(tenant, assignment, conversation, connectors, clock, knowledge)
        result = pack.handle(domain_event, ctx)            # → replies + actions + nuevo estado
        with transaction.atomic():
            persist_state(result.state)
            intents = plan_sends(result.replies, channel)  # fusión de textos, ventana de 24 h, degradación
            apply_actions(result.actions)                  # programar/cancelar tareas, handoff, avisar al dueño
            record_execution(event, result, timings, costs)
        enqueue_sends(intents)
```

Todo pack recibe **solo** un `PackContext` (§5.3) y devuelve un `HandleResult`. **Nunca toca la BD ni las APIs directamente.**

---

### 8. Inteligencia artificial

> **En pocas palabras.** La IA se usa para cuatro cosas concretas: contestar preguntas sobre el negocio con su documento de información, transcribir audios, entender fechas escritas a mano ("el jueves por la tarde") y clasificar mensajes. **Nunca** decide reservas, precios ni horarios, y nunca charla de temas ajenos al negocio (Meta lo prohíbe en WhatsApp).

#### 8.1 Usos permitidos

| Uso | Dónde | Garantías |
|---|---|---|
| FAQ con knowledge | Motor de conversación (texto libre no entendido) y bloque `faq` | Responde solo desde el knowledge; si no está → respuesta segura ("No tengo esa información; te paso con el equipo") + registro de pregunta sin respuesta |
| Transcripción de audios | Antes de procesar un mensaje `media` de tipo audio | El texto entra en el flujo normal; si falla → "¿Me lo puedes escribir?" |
| Extracción | Bloques `ask_date`, `ask_stop` | Salida con esquema; validación por código; **confirmación explícita** del usuario |
| Clasificación | Derivar a humano por urgencia o enfado; gimnasio | Lista cerrada de etiquetas; ante la duda → menú |
| Resúmenes internos | Aviso de derivación, informe mensual | Solo para el dueño |

#### 8.2 Guardrails

- System prompt fijo por plataforma + datos del negocio: "eres el asistente de <negocio>; responde solo sobre <negocio> usando la información proporcionada; si no está, dilo".
- Filtro de salida (portado de `guardrails.py`): longitud máxima, sin URLs no presentes en el knowledge, sin precios que no aparezcan en el knowledge o la config.
- **Minimización de datos:** sin teléfonos en el prompt; el nombre solo si hace falta; **nunca datos de salud**.
- Temperatura baja; historial limitado a los últimos 6 turnos.

#### 8.3 Cuotas, coste y proveedor

- Cada llamada registra tokens y coste en `execution` y `usage_daily`.
- **Cuota mensual por tenant** (`tenant.llm_monthly_token_quota`); al superarla → respuesta segura y aviso a ops.
- Proveedor y modelo **por configuración** (`connector_binding` de la categoría `llm`; por defecto el de plataforma). Los ids de modelo nunca en el código (Google y otros retiran modelos con frecuencia).
- **Evals:** cada pack con IA tiene `evals/` (30–50 preguntas con el comportamiento esperado: responder / respuesta segura). Se ejecutan en CI con un LLM falso para la estructura, y **manualmente o en un job nocturno de staging** contra el modelo real al cambiar de modelo o de prompt.

#### 8.4 Cumplimiento

- **Aviso de IA** (AI Act art. 50, desde el 2/8/2026): el primer mensaje de cada conversación nueva incluye el texto `texts.ai_disclosure` ("Soy el asistente automático de X. Escribe HUMANO para hablar con una persona"). Lo inserta el motor y **no se puede desactivar**.
- **Política de WhatsApp:** prohibido el chat de propósito general; el system prompt rechaza los temas ajenos al negocio.
- Proveedores de IA = subencargados de tratamiento (en el DPA).

---

### 10. Fiabilidad: que no se pierda ni se duplique nada

> **En pocas palabras.** Internet falla, los servidores se reinician y Meta a veces envía el mismo aviso dos veces. El sistema está hecho para que en esos casos **ningún mensaje se pierda** (todo se guarda antes de procesarse) y **nada se haga dos veces** (cada operación importante tiene una "clave" que impide repetirla).

#### 10.1 Entrada idempotente

```
POST /webhooks/<canal>/<channel_id>
  1. verificar firma                         → 401 sin guardar nada si falla
  2. resolver canal → tenant                 → 404 si no existe o está dado de baja (pero 200 para Meta, ver nota)
  3. BEGIN
       INSERT inbound_event … ON CONFLICT (channel_id, provider_event_id) DO NOTHING
       si se insertó: procrastinate.defer(process_event, queue="inbound", lock=f"conv:{channel_id}:{contact}")
     COMMIT
  4. 200 OK
```

Nota: a Meta y Telegram se les devuelve **200 aunque el canal esté pausado** (si no, reintentan durante días); el evento se guarda con estado `ignored`.

#### 10.2 Colas y trabajos

| Cola | Trabajos | Candado (lock) | Reintentos |
|---|---|---|---|
| `inbound` | `process_event(event_id)` | `conv:<channel>:<contact>` → orden por conversación | 5, backoff exponencial (2 s → ~1 min) |
| `outbound` | `send_message(intent_id)` | `out:<conversation>` → se envían en orden | Según el error (§10.3) |
| `scheduled` | `run_task(scheduled_task_id)`, periódicas | `task:<key>` | 3 |
| `heavy` | `import_bus_excel`, `monthly_report`, `retention_cleanup`, `export_tenant` | `heavy:<tenant>` → una a la vez por tenant | 2 |

- **Una sola copia de worker atiende las 4 colas al principio.** Se separan en servicios distintos cuando haga falta (§20), sin tocar código.
- Trabajo que agota sus reintentos → estado `failed` en Procrastinate → visible en **Consola → Trabajos fallidos**, con botón "Reintentar" → alarma (§18.4).
- Los trabajos reciben **solo ids**, nunca datos personales en los argumentos.

#### 10.3 Envío de mensajes

```
send_message(intent_id):
  intent.status == sent              → terminar (ya se envió)
  intent.status == unknown           → no reenviar automáticamente (ver abajo)
  status = sending; attempts += 1; COMMIT
  respuesta del proveedor:
    OK                → status = sent, provider_message_id = wamid; crear message(out)
    4xx (no 429)      → status = failed (sin reintento); si es el token → avisar a ops
    429 / 5xx         → status = pending; reintento con backoff (el proveedor NO lo aceptó)
    timeout / red     → status = unknown; tarea de conciliación en 5 min:
                          ¿llegó un status webhook con ese mensaje? → sent
                          si no → alerta en la consola; reintento manual o automático solo si el mensaje
                                  es inocuo de repetir (configurable por tipo de reply)
```

Garantía: **al menos una vez + idempotencia**. Nunca se promete "exactamente una vez".

#### 10.4 Efectos externos idempotentes

- `ctx.idem.key("booking:create")` → `"<tenant>:<conversation>:<event_id>:booking:create"`: estable si el mismo evento se reprocesa.
- El registro de conectores, en las operaciones que escriben: busca la clave en `side_effect`; si está `done`, devuelve el resultado guardado sin llamar al sistema externo; si no, la registra como `started`, llama y la marca como `done` con el resultado.
- Google Calendar, además, usa un id de evento derivado de la clave (doble protección).

#### 10.5 Tareas programadas

- **Programar** (acción `ScheduleTask(key, task_name, run_at, payload)`): upsert en `scheduled_task` por `(tenant, key)` + trabajo de Procrastinate con `schedule_at=run_at`. Reprogramar con la misma clave cancela el trabajo anterior y crea uno nuevo.
- **Cancelar** (`CancelTask(key)`): marca `cancelled` y borra el trabajo pendiente. Si no existe, no falla.
- **Al ejecutarse**: comprueba que `scheduled_task.status == scheduled` (si se canceló justo antes, no hace nada), carga el tenant y llama a `pack.handlers["task:<name>"]`. **El handler verifica que la entidad sigue siendo válida** (la cita no se ha cancelado ni movido) antes de actuar.
- **Periódicas**: cada `Periodic` del pack se registra como tarea periódica de Procrastinate de plataforma, que **reparte**: crea un trabajo por cada tenant activo con ese pack (nunca un trabajo gigante para todos).
- Recuperación: si el worker estaba caído a la hora programada, el trabajo se ejecuta al volver (sigue en la cola). Los trabajos con más de N horas de retraso (por tarea: un recordatorio de una cita que ya pasó) se descartan y se registran.

#### 10.6 Concurrencia del dueño y del bot

Si una conversación está en modo `human`, el motor no responde (solo guarda y reenvía al dueño). `human_until` (por defecto 12 h) devuelve la conversación al bot automáticamente, y "Devolver al bot" lo hace al momento.

---

### 11. Seguridad y secretos

> **En pocas palabras.** Las contraseñas y tokens de los clientes se guardan **cifrados** con una clave que custodia AWS (KMS) y que nunca sale de ahí. Todos los avisos que llegan de fuera se comprueban con una firma. La consola tiene doble factor. Los logs nunca contienen tokens ni teléfonos completos.

#### 11.1 Credenciales de clientes (cifrado por sobre)

```
Guardar:  KMS.GenerateDataKey(clave maestra) → clave de datos (en claro + cifrada)
          cifrar el secreto con AES-256-GCM usando la clave en claro → ciphertext
          guardar ciphertext + clave de datos cifrada + kms_key_id;  olvidar la clave en claro
Usar:     KMS.Decrypt(clave de datos cifrada) → clave en claro → descifrar → usar en memoria → olvidar
```

- Interfaz `CredentialCipher` con dos implementaciones: `KmsCipher` (staging y prod) y `LocalCipher` (desarrollo, clave en `.env`).
- Caché en memoria de las claves de datos descifradas, máx. 5 min (menos llamadas a KMS).
- Cada `Decrypt` queda registrado en CloudTrail (auditoría de quién accede a secretos).
- Rotación: KMS rota la clave maestra cada año automáticamente; los tokens de los clientes se cambian desde la consola (`rotated_at`).

#### 11.2 Secretos de plataforma

En **SSM Parameter Store (SecureString)**, o Secrets Manager si necesitan rotación automática. Se inyectan como variables de entorno en las tareas ECS mediante la definición de tarea (nunca en la imagen ni en Git): `DJANGO_SECRET_KEY`, `DATABASE_URL_*`, `OWNER_BOT_TOKEN`, `SENTRY_DSN`, claves de los proveedores de IA de plataforma.

#### 11.3 Superficie de ataque

| Punto | Protección |
|---|---|
| Webhooks | Firma (HMAC Meta / secret_token Telegram / HMAC propio en formularios y webhooks genéricos) **antes de nada**; tamaño máximo del cuerpo; rate limit por IP |
| Web chat | Rate limit por IP y sesión; CORS; longitud máxima del mensaje |
| Consola y admin | Solo en `consola.<dominio>`; **regla del ALB con lista de IPs permitidas** (casa, móvil vía VPN o Tailscale); login de Django + **2FA (django-otp)**; sesiones cortas; cada acción en `audit_event` |
| Red | RDS en subred privada (solo accesible desde las tareas); tareas con security group que solo acepta tráfico del ALB; ALB solo en 443 (80 redirige) |
| Dependencias | `uv.lock`; Dependabot; `pip-audit` en CI; imagen base mínima, actualizada mensualmente |
| Logs | Filtro que enmascara tokens, `Authorization` y teléfonos (últimos 4 dígitos) |
| Acceso a AWS | Sin usuarios IAM con claves; acceso por IAM Identity Center + MFA; CI por OIDC; roles de tarea con permisos mínimos (S3 de su bucket, KMS de su clave, SSM de su ruta) |
| Contenedores | Usuario no root, sistema de ficheros de solo lectura (salvo `/tmp`) |

#### 11.4 Datos personales

- Se registran en `message` solo los campos necesarios. Los datos de salud **nunca** en el texto libre de los packs de clínica ni en prompts.
- Exportación y borrado por tenant y por contacto (derechos RGPD): comandos `export_tenant`, `forget_contact`.

---

### Security boundaries

**Clasificación de datos**

- **Datos personales** (teléfonos, nombres, citas, contenido de mensajes): se minimizan (§8.2, §11.4), se cifran en reposo y se borran o anonimizan según §9.3.
- **Credenciales** (tokens de Meta, Telegram y Google, `app_secret`, claves de IA): cifradas con KMS en la base de datos (§11.1) o en SSM (§11.2). Nunca en claro.
- **Datos de salud**: fuera del MVP. No entran en prompts ni en packs (§8.2, §11.4).
- **Uso y coste** (`usage_daily`): agregados, sin datos personales (§9.1).

**Qué cruza cada frontera**

Flujo: webhooks → web → cola → worker → conectores externos.

| Frontera | Qué cruza | Protección |
|---|---|---|
| Internet → webhooks (`web`) | Payload del proveedor (mensajes y avisos de estado) | Firma verificada antes de guardar nada (§10.1, §11.3) |
| `web` → cola (Postgres) | `InboundEvent` con su payload; los trabajos reciben solo ids (§10.2) | Idempotencia por `provider_event_id` (§10.1) |
| Cola → worker | Id del evento; el worker carga los datos del tenant | `SET LOCAL app.tenant_id` y RLS (§9.2) |
| Worker → conectores externos (Google, IA, Meta) | Solo los datos necesarios; sin teléfonos en prompts (§8.2) | Credenciales descifradas solo en memoria; timeouts y reintentos (§7.4) |
| Consola → datos | Solo con rol `operator` (`app_operator`) | Login con 2FA, IP permitida, cada acción auditada (§9.2, §11.3) |

**Qué nunca puede cruzar**

- Credenciales en claro: ni en Git, ni en logs, ni en imágenes, ni en respuestas de la API (§11.3).
- Datos de un tenant en otro: RLS y claves foráneas compuestas `(tenant_id, x_id)` (§9.2).
- Teléfonos completos y tokens en logs (§11.3).
- Datos de salud en texto libre, prompts o packs (§8.2, §11.4).

**Cifrado**

- **En tránsito:** HTTPS en el ALB (80 redirige a 443) (§11.3).
- **En reposo:** RDS cifrado y buckets S3 cifrados (§17.4). Las credenciales de clientes se cifran con AES-256-GCM usando una clave de datos que KMS envuelve; KMS rota la clave maestra cada año (§11.1).

**Modelo de confianza entre módulos**

- Los packs acceden a los datos solo a través de la API de packs (`PackContext`); no tocan la base de datos ni las APIs directamente (§4.3, §5.3).
- `core/` no importa Django, `platform/` ni `packs/`; un pack no importa otro pack (§4.2).
- Los conectores se llaman solo a través del proxy del registro, que resuelve binding y credencial (§7.1).
- La consola y el Django Admin solo los usa el rol operador (§9.2, §13).
- Las operaciones que escriben en sistemas externos son idempotentes (§10.4).

---

### 12. Configuración: qué vive dónde

> **En pocas palabras.** Hay tres sitios para guardar "ajustes" y cada cosa tiene uno solo. Así nunca hay dudas de dónde cambiar algo ni de si un cambio necesita desplegar.

| Dónde | Qué | Cómo se cambia | ¿Despliegue? |
|---|---|---|---|
| **Repositorio** | Código, packs, bloques, recetas, plantillas de WhatsApp por defecto, manifests de conectores, migraciones, infraestructura (CDK) | PR + CI + revisión | Sí |
| **Base de datos** | Tenants, canales, bindings, credenciales, config de negocio, knowledge, asignación de pack/receta, plantillas del tenant, contratos | Consola, Django Admin, bot del dueño (con validación y auditoría) | **No** |
| **Entorno** (SSM → variables de entorno) | Lo que cambia entre local/staging/prod: URLs, secretos de plataforma, `META_GRAPH_VERSION`, modelo de IA por defecto, tamaños de pool, niveles de log | CDK / consola de AWS | Reinicio de tareas (lo hace el despliegue) |

**Variables de entorno principales:**

```text
ENVIRONMENT=local|staging|prod
DJANGO_SECRET_KEY, DJANGO_ALLOWED_HOSTS, DJANGO_DEBUG=false
DATABASE_URL_RUNTIME, DATABASE_URL_OPERATOR, DATABASE_URL_OWNER (solo en la tarea migrate)
PUBLIC_API_BASE_URL=https://api.<dominio>
CREDENTIAL_CIPHER=kms|local, KMS_KEY_ID, LOCAL_CIPHER_KEY (solo local)
S3_BUCKET_MEDIA, AWS_REGION
META_GRAPH_VERSION=v2x.0
OWNER_BOT_TOKEN, OWNER_BOT_WEBHOOK_SECRET, OPS_TELEGRAM_CHAT_ID
LLM_DEFAULT_ADAPTER, LLM_DEFAULT_MODEL, <PROVEEDOR>_API_KEY
SENTRY_DSN, LOG_LEVEL
WORKER_QUEUES=inbound,outbound,scheduled,heavy   WORKER_CONCURRENCY=8
```

---

### 13. Consola y Django Admin

> **En pocas palabras.** Una web interna (`consola.<dominio>`) para ver si todo va bien, dar de alta clientes y resolver problemas, sin tocar la base de datos a mano. El Django Admin queda para los casos raros.

#### 13.1 Tecnología

Vistas de Django con plantillas + **HTMX** (actualizaciones parciales sin escribir JavaScript) + una hoja de estilos sencilla (Pico.css o similar). Sin frontend separado ni SPA. Autenticación de Django + 2FA. Permisos: `operator` (todo) y, en el futuro, `tenant_owner` (solo su tenant, vía RLS con `app_runtime`).

#### 13.2 Pantallas

| Pantalla | Contenido |
|---|---|
| **Inicio (salud)** | Estado general (verde/ámbar/rojo); trabajos pendientes y antigüedad del más antiguo por cola; errores en la última hora; conectores caídos; envíos `unknown`/`failed`; plantillas rechazadas |
| **Tenants** | Lista con estado, pack, canales, último mensaje, mensajes y coste de Meta del mes, salud de integraciones |
| **Ficha de tenant** | Config de negocio (formulario generado del esquema, con historial y diff), knowledge (editor markdown con versiones), canales, bindings, plantillas, tareas programadas, contrato, botón **"Comprobar salud"**, botones Pausar/Reanudar |
| **Asistente de alta** | 1) datos básicos → 2) pack → 3) config (formulario del esquema) → 4) canales (WhatsApp: modo, ids, secretos; Telegram: token) → 5) conectores (formularios de los manifests) → 6) knowledge desde la plantilla → 7) plantillas de WhatsApp → 8) comprobar salud → 9) activar + enlace de vinculación del bot del dueño |
| **Conversaciones** | Búsqueda por tenant, teléfono o fecha → hilo con mensajes entrantes y salientes, ejecuciones, errores y `trace_id` |
| **Trabajos** | Pendientes, en curso y fallidos por cola; reintentar o cancelar |
| **Tareas programadas** | Por tenant: próximas, ejecutadas, canceladas |
| **Uso y costes** | Por tenant y mes: mensajes por categoría, coste de Meta estimado, tokens de IA, ejecuciones |
| **Packs y recetas** | Versiones registradas, qué tenants usan cada receta, bloques sin uso |
| **Auditoría** | `audit_event` filtrable |

#### 13.3 Comprobación de salud de un tenant

Botón en la ficha y tarea diaria automática. Comprueba:

- canales: webhook verificado, token válido (llamada de lectura a la API), plantillas aprobadas;
- conectores: el `health_check` de cada binding (p. ej. leer el calendario);
- config: valida contra el esquema actual del pack;
- receta: válida y con todos sus bloques disponibles;
- escenarios: ejecuta los del pack **con la config y la receta del tenant** y conectores mock (en el worker, cola `heavy`).

El resultado se guarda y aparece en verde, ámbar o rojo por tenant.

---

### 14. Entornos y desarrollo local

> **En pocas palabras.** Hay tres entornos: tu portátil (local), uno de pruebas en AWS (staging) y el real (prod). Los tres ejecutan **la misma imagen**; solo cambia la configuración. **Nunca hay datos reales fuera de prod.**

| | Local | Staging | Prod |
|---|---|---|---|
| Dónde | Tu portátil, Docker Compose | Cuenta AWS de staging | Cuenta AWS de prod |
| BD | Postgres 17 en un contenedor | RDS db.t4g.micro, Single-AZ | RDS db.t4g.small (→ Multi-AZ) |
| Cifrado | `LocalCipher` | KMS | KMS |
| Canales | Web chat, bot de Telegram de pruebas, número de prueba de Meta (vía túnel) | Número de prueba de Meta, bot de Telegram de staging | Reales |
| Datos | Semilla (`seed_demo`) | Ficticios | Reales |
| Despliegue | `docker compose up` | Automático en cada merge a `main` | Manual con aprobación |
| Coste | 0 | ~15–30 $/mes (se apaga fuera de horario) | §17.6 |

#### 14.1 Desarrollo local

```bash
uv sync                          # dependencias
docker compose up -d db          # Postgres
make migrate seed                # esquema + tenants de demo (salon, bus, gym) con conectores mock
make dev                         # web (runserver) + worker (procrastinate) con recarga automática
make tunnel                      # cloudflared/ngrok → URL pública para webhooks de Telegram/Meta de pruebas
make test                        # todos los tests;  make test-fast  (sin Postgres: core + packs)
make scenarios PACK=salon        # escenarios de un pack
```

- `compose.yaml` local: `db` (Postgres 17), `web`, `worker` y `mailpit` (para ver los emails enviados).
- Hay datos de demo para probar sin credenciales: los conectores `mock` devuelven huecos de calendario y respuestas de IA predefinidas.

---

### 16. Construcción y despliegue

> **En pocas palabras.** Cada vez que se aprueba un cambio, se construye **una imagen** (un paquete con todo el programa) y se despliega primero en staging automáticamente. Para producción pulsas un botón. AWS arranca las copias nuevas, comprueba que funcionan y solo entonces apaga las viejas: **no hay cortes**. Si las nuevas fallan, vuelve solo a la versión anterior.

#### 16.1 La imagen

- Un único `Dockerfile` multi-etapa (dependencias con `uv` → imagen final mínima), **ARM64** (Graviton).
- Etiqueta = SHA del commit. Se publica en **ECR** (cuenta de prod, compartida con staging mediante permisos entre cuentas).
- La misma imagen para `web`, `worker` y `migrate` (cambia solo el comando).
- Los estáticos (CSS de la consola) se sirven con WhiteNoise desde la propia imagen.

#### 16.2 Pipeline (GitHub Actions)

```
on: pull_request  → job "ci": puertas §15.4

on: push a main   →
  1. build: construir la imagen ARM64 → push a ECR (tag = sha)
  2. deploy-staging (automático):
       a. asumir el rol de despliegue de staging (OIDC, sin claves guardadas)
       b. ejecutar la tarea ECS "migrate" con la imagen nueva (migrate + sync_packs) y esperar a que termine bien
       c. actualizar los servicios web y worker con la nueva definición de tarea
       d. esperar a que el despliegue se estabilice (o se revierta solo)
       e. smoke test: /health + conversación sintética por Telegram
  3. deploy-prod (manual: "environment: production" con aprobación requerida):
       mismos pasos a–e contra la cuenta de prod
```

#### 16.3 Cómo se despliega sin cortes

- **web**: despliegue progresivo de ECS con `minimumHealthyPercent=100`, `maximumPercent=200`. Arranca las tareas nuevas → el ALB comprueba `/health` → les envía tráfico → retira las viejas tras 30 s de *draining* (dejan de recibir peticiones nuevas y terminan las que tienen).
- **worker**: al pararse recibe SIGTERM → Procrastinate deja de coger trabajos nuevos y termina los que tiene → `stopTimeout=120 s`. Los pendientes siguen en la BD y los recoge la tarea nueva.
- **Deployment circuit breaker con rollback automático**: si las tareas nuevas no pasan los health checks, ECS vuelve solo a la versión anterior.
- **Migraciones**: se ejecutan **antes** de actualizar los servicios. Como son compatibles con la versión anterior (§9.4), el código viejo sigue funcionando durante el despliegue.

#### 16.4 Vuelta atrás (rollback)

- **Automática**: el circuit breaker de ECS.
- **Manual**: re-ejecutar `deploy-prod` con el SHA anterior (botón en GitHub Actions). Las migraciones no se revierten (expand/contract las hace innecesarias).
- **Datos**: restauración a un momento concreto (PITR, §19.3), solo en caso de corrupción.

#### 16.5 Cuándo y cómo se despliega

- Staging: en cada merge.
- Prod: cuando quieras, **preferiblemente fuera de las horas punta de los clientes** (antes de las 9:00 o después de las 21:00). Con los despliegues sin corte no es obligatorio, pero reduce el riesgo.
- **Cambios de config de clientes: nunca necesitan despliegue.**
- Cambios de infraestructura: `cdk diff` en el PR (como comentario) → `cdk deploy` manual tras la aprobación, primero en staging.

---

### 17. Infraestructura en AWS

> **En pocas palabras.** Usamos servicios gestionados de Amazon: **ECS Fargate** ejecuta nuestro programa sin que tengamos que gestionar servidores (sube o baja el número de copias según la carga), **RDS** es una base de datos PostgreSQL que AWS mantiene (copias de seguridad, parches, réplica en otra zona), y el **ALB** es la puerta de entrada con HTTPS. Todo se define en código (CDK) para poder recrearlo idéntico. Empieza costando ~95 €/mes en producción y crece solo si hace falta.

#### 17.1 Cuentas y acceso

```
AWS Organizations (cuenta de gestión: solo facturación y organización, sin recursos)
 ├─ cuenta "staging"   → todo el entorno de pruebas
 └─ cuenta "prod"      → producción + ECR (registro de imágenes compartido)
Acceso humano: IAM Identity Center (SSO) + MFA. Sin usuarios IAM con claves.
CI: rol por cuenta asumible solo desde GitHub Actions de este repo (OIDC).
Presupuestos (AWS Budgets) con alertas por email al 50/80/100 % en cada cuenta.
```

#### 17.2 Región

**eu-south-2 (España, Aragón)** [VERIFICAR antes de crear nada]: que estén disponibles ECS Fargate (ARM), RDS PostgreSQL 17, ECR, KMS, SSM, CloudWatch, ALB, ACM y SES. Precios similares a Irlanda; argumento comercial "datos en España". **Plan B: eu-west-1 (Irlanda)**, la región más completa. Si SES no está en eu-south-2, el email se envía con SES de eu-west-1 o con Brevo (no afecta a los datos).

#### 17.3 Red

```
VPC (2 zonas de disponibilidad)
 ├─ subredes públicas (2): ALB + tareas ECS (con IP pública, security group cerrado)
 └─ subredes privadas (2): RDS (sin acceso a internet)

Security groups:
  sg-alb    : entrada 443/80 desde internet
  sg-tasks  : entrada 8000 SOLO desde sg-alb; salida a internet (Meta, Google, IA…)
  sg-db     : entrada 5432 SOLO desde sg-tasks
```

**[DECIDIDO] Sin NAT Gateway al principio** (cuesta ~33 $/mes + tráfico): las tareas salen a internet con su IP pública (3,65 $/mes cada una) y no aceptan conexiones salvo desde el ALB. Se pasa a subredes privadas + NAT cuando haya más de ~8 tareas o lo exija un cliente (§20).

#### 17.4 Servicios y tamaños iniciales

| Recurso | Configuración inicial (prod) | Staging |
|---|---|---|
| **ECS cluster** | Fargate, ARM64 | Igual |
| Servicio `web` | 0,5 vCPU / 1 GB, 1 tarea (mín. 1, máx. 4), autoescala por CPU 60 % y por peticiones por tarea | 0,25 vCPU / 0,5 GB, 1 tarea |
| Servicio `worker` | 0,5 vCPU / 1 GB, 1 tarea (mín. 1, máx. 6), autoescala por antigüedad del trabajo más antiguo (> 30 s → +1) | 0,25 vCPU / 0,5 GB |
| Tarea `migrate` | 0,5 vCPU / 1 GB, bajo demanda | Igual |
| **ALB** | 1, listener 443 (certificado ACM gratuito), reglas por host: `api.`, `consola.` (con filtro de IP origen) | Igual |
| **RDS PostgreSQL 17** | db.t4g.small (2 GB), 20 GB gp3 con crecimiento automático, cifrado, Single-AZ → **Multi-AZ** con 3–5 clientes de pago; backups 14 días + PITR; protección contra borrado | db.t4g.micro, 7 días de backup, se puede detener |
| **S3** | Bucket `media` (cifrado, privado, versionado, ciclo de vida 90 días), bucket `exports` (7 días) | Igual |
| **KMS** | 1 clave para credenciales + la de RDS/S3 gestionada por AWS | Igual |
| **ECR** | 1 repositorio, regla de ciclo de vida (guardar las últimas 30 imágenes) | (usa el de prod) |
| **CloudWatch** | Logs con retención de 30 días; métricas propias; alarmas → SNS | Retención de 7 días |
| **Route 53** | Zona del dominio; `api.`, `consola.`, `widget.` (futuro) | `*.staging.<dominio>` |
| **SES** | Dominio verificado (SPF, DKIM, DMARC) | Sandbox |

**Detalles importantes:**

- **Conexiones a la BD:** sin RDS Proxy (Procrastinate usa `LISTEN/NOTIFY`, incompatible con multiplexar conexiones). Pool pequeño por tarea (`CONN_MAX_AGE` + máx. ~10 conexiones por tarea). db.t4g.small admite ~200 conexiones, de sobra.
- **Health check:** `/health` comprueba que el proceso responde y que hay conexión a la BD (sin llamar a servicios externos).
- **Fargate sin acceso SSH:** para depurar se usa **ECS Exec** (consola dentro del contenedor, auditada), solo con tu rol.

#### 17.5 Infraestructura como código (CDK en Python)

```
infra/
  app.py                       # define staging y prod con sus parámetros
  stacks/
    network_stack.py           # VPC, subredes, security groups
    data_stack.py              # RDS, KMS, S3, parámetros SSM
    registry_stack.py          # ECR (solo prod) + permisos para staging
    app_stack.py               # cluster ECS, definiciones de tarea, servicios, ALB, autoescalado, DNS
    observability_stack.py     # alarmas, SNS, Lambda de alertas a Telegram, dashboard
    ci_stack.py                # proveedor OIDC de GitHub + roles de despliegue
  config/
    staging.py  prod.py        # tamaños, número de tareas, Multi-AZ sí/no, dominios
```

- Cambiar de Single-AZ a Multi-AZ, o de 1 a 2 tareas web = cambiar un valor en `prod.py` → PR → `cdk deploy`.
- Recursos con datos (RDS, buckets, clave KMS) con **protección contra borrado** y `RemovalPolicy.RETAIN`.

#### 17.6 Costes [ESTIMACIÓN con precios de referencia de 2026; verificar en la calculadora de AWS para la región elegida]

| Pieza | Prod inicial | Prod con alta disponibilidad (3–5 clientes de pago) |
|---|---:|---:|
| Fargate ARM (web + worker) | ~32 $ | ~48 $ (2 web) |
| IPv4 públicas (tareas + ALB) | ~15 $ | ~18 $ |
| ALB | ~20 $ | ~20 $ |
| RDS db.t4g.small + almacenamiento + backups | ~30 $ | ~58 $ (Multi-AZ) |
| KMS, SSM, CloudWatch, ECR, S3, Route 53, SES | ~8 $ | ~10 $ |
| **Total prod** | **~105 $/mes (~95 €)** | **~155 $/mes (~140 €)** |
| Staging (apagado fuera de horario) | ~15–30 $/mes | ~15–30 $/mes |

**Para reducirlo:** créditos de cuenta nueva (100–200 $); **AWS Activate Founders** (1.000 $, tras el alta); **no crear prod hasta el primer cliente real**; Savings Plans (−20–40 %) cuando el uso sea estable; staging apagado de noche y en fines de semana (tareas a 0, RDS detenido).

#### 17.7 Fases de infraestructura

| Fase | Qué hay |
|---|---|
| **Desarrollo** | Local + staging bajo demanda (créditos) |
| **Primer cliente real** (migración de la peluquería) | Prod inicial (Single-AZ, 1 web + 1 worker) |
| **3–5 clientes de pago** | RDS Multi-AZ + 2 tareas web en 2 zonas |
| **20+ clientes o caídas caras** | Subredes privadas + NAT, WAF, worker `heavy` separado, copia de backups a otra región, Savings Plans |

---

### 18. Observabilidad y alertas

> **En pocas palabras.** Saber en todo momento si algo va mal **antes de que te llame un cliente**, y poder reconstruir cualquier conversación en minutos.

#### 18.1 Logs

- **JSON estructurado** a la salida estándar → CloudWatch Logs.
- Campos obligatorios: `ts`, `level`, `msg`, `env`, `service` (web/worker), `tenant_id`, `trace_id`, `event_id`, `conversation_id`, `job`.
- `trace_id` = id del `inbound_event` que originó todo; se propaga a ejecuciones, envíos, llamadas a conectores y logs. Buscar por `trace_id` muestra la historia completa.
- Sin datos sensibles (§11.3).

#### 18.2 Errores

**Sentry** (plan gratuito al principio): excepciones de web y worker con el contexto (`tenant_id`, `trace_id`), y versiones por SHA de despliegue.

#### 18.3 Métricas

| Métrica | Origen | Uso |
|---|---|---|
| `queue_pending{queue}`, `queue_oldest_age_s{queue}` | Tarea periódica cada 60 s → CloudWatch | **Autoescalado del worker** y alarmas |
| `jobs_failed_total{queue}` | Ídem | Alarma |
| `webhook_latency_ms` (p50/p95) | Middleware → logs con formato de métrica (EMF) | Alarma |
| `send_status{status}` (sent/failed/unknown) | Ídem | Alarma |
| `connector_calls{category,adapter,result}`, `breaker_open` | Registro de conectores | Consola + alarma |
| `llm_tokens{tenant}`, `cost_eur{tenant,kind}` | `usage_daily` | Consola, cuotas, informe |
| CPU y memoria por servicio, conexiones de RDS, CPU de RDS, espacio libre | AWS (automático) | Alarmas y autoescalado |

#### 18.4 Alarmas → Telegram (ops) + email

| Alarma | Umbral inicial | Gravedad |
|---|---|---|
| `/health` del ALB sin tareas sanas | 1 min | 🔴 |
| `queue_oldest_age_s` inbound | > 60 s durante 3 min | 🔴 |
| `jobs_failed_total` | > 5 en 10 min | 🟠 |
| Envíos `failed` + `unknown` | > 3 en 15 min | 🟠 |
| Circuit breaker abierto (cualquier tenant) | 1 | 🟠 |
| Errores 5xx del ALB | > 1 % durante 5 min | 🟠 |
| RDS: CPU > 80 % / espacio libre < 20 % / conexiones > 80 % | 10 min | 🟠 |
| Plantilla de WhatsApp rechazada / calidad del número baja | 1 | 🟡 |
| Tenant > 800 mensajes de servicio en el mes | 1 | 🟡 (aviso de coste al cliente) |
| Despliegue revertido automáticamente | 1 | 🟠 |
| Presupuesto de AWS al 80 % | 1 | 🟡 |

Camino: alarma de CloudWatch → SNS → (email) + Lambda mínima que publica en el chat de ops de Telegram. **Las alarmas no dependen de que la plataforma esté viva** (si se cae la web, el aviso llega igualmente).

#### 18.5 Panel

Dashboard de CloudWatch (definido en CDK) con las métricas de §18.3 + la pantalla de inicio de la consola (§13.2). Uptime externo (Better Stack o UptimeRobot, gratis) contra `https://api.<dominio>/health` desde fuera de AWS.

---

### 19. Operación técnica

> **En pocas palabras.** Cómo dar de alta y de baja a un cliente, hacer y probar copias de seguridad, y qué hacer cuando algo falla.

#### 19.1 Alta de un tenant

1. **Consola → Asistente de alta** (§13.2). Si el cliente necesita receta propia: PR con `tenants/<slug>/recipe.yaml` antes de activarlo.
2. WhatsApp (`client_app`): el cliente nos da acceso a su Meta App → introducimos `app_id`, `app_secret`, token, `waba_id` y `phone_number_id` → la consola muestra la URL del webhook y el `verify_token` para configurarlos en su app → se suscribe el campo `messages`.
3. Telegram del dueño: enlace de vinculación.
4. Plantillas: enviadas a aprobación desde la consola.
5. **Comprobar salud** en verde.
6. Prueba real con el dueño (3 reservas de prueba).
7. Activar. Se registra en `audit_event`.

#### 19.2 Baja de un tenant

Comando o botón `offboard_tenant <slug>`:

1. Estado `offboarded` → los webhooks se aceptan y se ignoran.
2. Cancelar las tareas programadas.
3. `export_tenant` (si el contrato lo prevé) → zip en S3 `exports` con un enlace temporal.
4. Borrado de datos personales y credenciales (se conservan las métricas agregadas y la auditoría).
5. Checklist manual: el cliente revoca el acceso en su Meta App; borrar el webhook de Telegram; verificar.
6. `audit_event`.

#### 19.3 Backups y restauración

| Qué | Cómo | Retención |
|---|---|---|
| RDS | Backups automáticos + **PITR** (restaurar a cualquier momento de los últimos 14 días) | 14 días |
| RDS (largo plazo) | Snapshot semanal copiado a otra región [FUTURO en la fase de 20+ clientes] | 3 meses |
| S3 | Versionado | Según el ciclo de vida |
| Configuración del tenant | `config_history` + snapshots en el repositorio (§15.3) | Indefinida |
| Infraestructura | CDK en Git | — |

**Simulacro de restauración mensual** (tarea programada en prod): restaurar el último backup en una instancia temporal de la cuenta de prod → script de comprobación (conteos por tenant, últimas filas) → borrar la instancia → resultado en el chat de ops. Objetivos: **RPO ≤ 5 min** (PITR), **RTO ≤ 2 h**.

#### 19.4 Runbooks (en `docs/runbooks/`)

1. El bot no responde a nadie.
2. El bot no responde a un tenant concreto (token caducado, webhook desconfigurado, número con calidad baja).
3. La cola se acumula.
4. Un conector caído (Google, IA).
5. Envíos en estado `unknown`.
6. Restaurar la BD (PITR).
7. Rollback de un despliegue.
8. Incidente de seguridad o fuga de datos (pasos y plazos de aviso al cliente).
9. Rotar credenciales de un tenant.
10. Migración del webhook de un cliente existente (y vuelta atrás).

---

### 20. Escalado

> **En pocas palabras.** Al principio una copia de cada cosa basta de sobra. Si crece la carga, AWS añade copias solo. Esta tabla dice qué vigilar y qué cambiar en cada caso, **sin rediseñar nada**.

**Capacidad de partida [ESTIMACIÓN]:** 50 peluquerías ≈ 75.000 mensajes al mes, con picos de 1–2 por segundo. Un worker con concurrencia 8 procesa 20–50 trabajos por segundo → **margen de 10 a 50 veces**.

| Señal | Acción | Coste/esfuerzo |
|---|---|---|
| `queue_oldest_age_s` > 30 s sostenido | El autoescalado añade workers (máx. 6) | Automático |
| Trabajos `heavy` retrasan a `inbound` | Separar un servicio `worker-heavy` (misma imagen, `WORKER_QUEUES=heavy`) | Cambio en CDK |
| CPU de web > 60 % | El autoescalado añade tareas web | Automático |
| CPU de RDS > 70 % sostenido | Subir la clase (t4g.small → t4g.medium → m7g) | Unos minutos de corte (con Multi-AZ, ~1 min) |
| Muchas lecturas de la consola o informes | Réplica de lectura de RDS para la consola | Cambio en CDK + router de Django |
| Más de ~8 tareas | Subredes privadas + NAT (en lugar de IPs públicas) | Cambio en CDK |
| Postgres como cola al límite (muy lejano: miles de trabajos/s) | Cambiar la cola a **SQS** detrás de la misma interfaz, con patrón *outbox* | Proyecto acotado |
| Un tenant muy grande | Límite de concurrencia por tenant; en el extremo, worker dedicado con cola propia | Configuración |
| Límites de Meta (por número o por usuario) | Control de ritmo en `outbound` (ya previsto) | — |

---

### 21. Estructura del repositorio

```
bots-platform/
├── pyproject.toml · uv.lock · manage.py · Dockerfile · compose.yaml · Makefile
├── AGENTS.md                         reglas para agentes (resumen de este documento)
├── design.md                         este documento
├── NEGOCIO.md                        negocio
├── src/
│   ├── config/                       settings/{base,local,staging,prod}.py · urls.py · wsgi.py
│   ├── core/                         ── Python puro, sin Django ──
│   │   ├── domain/                   InternalMessage, eventos, Reply, Action, ConversationState
│   │   ├── packs/                    Pack, Periodic, PackContext, HandleResult, registro
│   │   ├── conversation/             motor de recetas, Step (Ask/Done/…), comandos globales
│   │   ├── blocks/                   bloques comunes  ·  blocks/booking/  bloques de reservas
│   │   ├── connectors/interfaces/    booking.py, llm.py, stt.py, sheets.py, email.py, http.py
│   │   ├── channels/                 capabilities, degradation, plan_sends (fusión, ventana 24 h)
│   │   └── testing/                  runner de escenarios, mocks programables
│   ├── platform/                     ── Django ──
│   │   ├── tenancy/  channels/{whatsapp,telegram,webchat,owner_bot}/  connectors/{registry,adapters/…}
│   │   ├── conversations/  events/  scheduling/  credentials/  usage/  audit/  console/
│   │   └── jobs.py                   app de Procrastinate + definición de tareas
│   └── packs/
│       ├── salon/   {pack.py, config.py, blocks/, handlers.py, recipes/, templates/, knowledge/, scenarios/, evals/}
│       ├── bus/     {…, models.py (tablas del pack), importer.py}
│       └── gym/
├── tenants/                          recetas propias, snapshots de config y escenarios por tenant
│   └── peluqueria-norte/ {recipe.yaml, config.snapshot.json, scenarios/}
├── tests/                            integration/ · isolation/ · contract/ (los unit y escenarios viven junto al código)
├── infra/                            CDK (§17.5)
├── docs/
│   ├── adr/                          ADR-001 … ADR-018
│   ├── packs/                        un documento por pack
│   ├── runbooks/
│   └── migracion.md
└── .github/workflows/                ci.yml · deploy.yml
```

---

### 22. Migración desde el código actual

> Detalle en `docs/migracion.md` (pendiente). Aquí, solo el mapa.

| Del repo actual | Destino | Cómo |
|---|---|---|
| `adapters/connectors/google_calendar/*` (incl. `compute_slots`) | `platform/connectors/adapters/google_calendar/` implementando `BookingConnector` | Portar + tests |
| `connectors/registry.py`, `circuit_breaker.py`, `catalog.py`, `errors.py` | `platform/connectors/` | Adaptar al proxy por tenant |
| `engine/degradation.py`, `outputs.py` | `core/channels/`, `core/domain/` | Portar casi igual |
| `adapters/channel/whatsapp.py` | `platform/channels/whatsapp/` (parse/render/send) | Separar la lógica pura de la vista |
| `adapters/channel/http_dev.py` + `chat_ui.html` | `platform/channels/webchat/` | Base del web chat |
| `adapters/connectors/gemini/` (+ `guardrails.py`), `mock_llm` | `platform/connectors/adapters/gemini/`, `core/testing/` | Implementar `LLMConnector` |
| `tests/runner/scenario_runner.py` + `tests/scenarios/*` | `core/testing/` + `packs/salon/scenarios/` | Ampliar el formato (§15.2) |
| `flows/peluqueria_flow.yaml` + knowledge | **Especificación** del pack `salon` (receta + bloques) | Reimplementar en Python |
| `engine/interpreter.py`, `flow.py` | ❌ Se retira tras portar `salon` | — |
| `control_plane/*`, `data_plane/main.py`, `cp_client.py`, APScheduler, SQLite | ❌ | Sustituidos por Django + RDS + Procrastinate |
| `legacy.md` | Se conserva (gotchas de Meta y Calendar, bugs ya resueltos) | Leer antes de portar |

Orden: etiquetar el estado actual (`v0-two-planes`) → esqueleto Django + `core/` → conectores y canales → pack `salon` con paridad de escenarios → borrar el código viejo.

---

### 23. Pendiente de verificar y decisiones abiertas

| # | Tema | Qué falta | Cuándo |
|---|---|---|---|
| 1 | Región eu-south-2 | Confirmar los servicios (ECS ARM, RDS PG 17, SES…) y los precios | Antes de crear la infraestructura |
| 2 | Python 3.14 | Confirmar que Django 5.2, Procrastinate y las dependencias lo soportan; si no, 3.13 | Esqueleto |
| 3 | Límites de envío de Meta | Rendimiento por número y límite por par usuario-negocio, para el control de ritmo de `outbound` | Canal WhatsApp |
| 4 | IA por defecto | Elegir proveedor y modelo con los evals del pack `salon` (Gemini / Mistral UE / Haiku) | Pack salon |
| 5 | Señal o prepago (bloque `request_deposit`) | Pasarela (Stripe Payment Links o Bizum vía TPV) | Cuando un cliente lo pida |
| 6 | Widget web de producción | CloudFront + S3, dominio por tenant o común | Fase de canales |
| 7 | Portal de cliente | Alcance y permisos | Cuando el bot del dueño no baste |
| 8 | Tech Provider / Embedded Signup | Implementar `platform_app` | Tras el alta y la verificación |
| 9 | Retención definitiva | Ajustar §9.3 al DPA firmado | Antes del primer cliente de pago |

---

## Module Contracts

### 3. Conceptos clave

> **En pocas palabras.** El vocabulario que usa el resto del documento y el código. Si un nombre aparece en el código, significa exactamente esto.

| Concepto | Qué es | Ejemplo |
|---|---|---|
| **Tenant** (cliente) | Un negocio que nos contrata. Todo dato pertenece a un tenant | "Peluquería Sur" |
| **Canal** | Una vía de comunicación de un tenant | El número de WhatsApp de la Peluquería Sur; el bot de Telegram del dueño |
| **Contacto** | Una persona que habla con el tenant por un canal | La clienta +34 6xx… |
| **Conversación** | El hilo entre un contacto y un tenant por un canal, con su estado (en qué paso está) | "Reservando: ha elegido corte, falta el día" |
| **Evento** | Algo que ocurre y hay que procesar | Mensaje recibido, formulario enviado, tarea programada que vence, webhook externo |
| **Pack** | El "programa" para un tipo de negocio: su lógica, su esquema de configuración y sus tests | `salon`, `bus`, `gym` |
| **Bloque** | Una pieza reutilizable de conversación o de lógica | "elegir servicio", "elegir hueco", "confirmar reserva" |
| **Recorrido** (journey) | Una tarea completa que hace el usuario, formada por bloques en orden | "Reservar", "Cancelar", "Mis citas" |
| **Receta** | Qué recorridos tiene un tenant y qué bloques (y con qué parámetros) los componen | La Peluquería B pide profesional; la A, no |
| **Variante** | Código Python específico de un tenant, cuando la receta no basta | Reglas de combinación de servicios muy particulares |
| **Config de negocio** | Los datos del tenant que usa el pack, validados con un esquema | Servicios, precios, duraciones, horario, textos, opciones on/off |
| **Knowledge** | El documento con la información del negocio que usa la IA para preguntas libres | Dirección, parking, política de cancelación |
| **Conector** | Un servicio para hablar con un sistema externo, por categoría | `booking` (Google Calendar), `llm` (Gemini/Mistral), `sheets` |
| **Binding** | Qué conector concreto usa un tenant para una categoría, con su config y credenciales | Peluquería Sur: `booking` → Google Calendar, calendario X |
| **Respuesta** (reply) | Lo que el pack quiere decir al contacto | Texto, botones, lista, plantilla, formulario |
| **Acción** | Lo que el pack quiere que ocurra además de responder | Programar recordatorio, avisar al dueño, derivar a humano |
| **Tarea programada** | Una acción para el futuro, con clave única | `reminder:booking_123` a las 10:00 de mañana |
| **Intención de envío** (send intent) | Un mensaje que hay que enviar, guardado antes de enviarlo | — |

---

### 5. Packs: cómo conviven los distintos negocios

> **En pocas palabras.** Cada tipo de negocio es un **pack**: una carpeta de Python con su lógica. Los packs se construyen con **bloques** reutilizables (piezas como "elegir día" o "confirmar reserva") que comparten entre ellos. Cada negocio concreto puede diferenciarse en tres niveles, de más fácil a más difícil:
> 1. **Config**: precios, horarios, textos, opciones on/off → lo cambia el dueño o tú en la consola, al momento.
> 2. **Receta**: qué pasos tiene cada recorrido y en qué orden (p. ej. una peluquería pide elegir profesional y otra no) → fichero en el repositorio, sin programar, pasa por tests.
> 3. **Variante**: código específico de ese negocio, solo cuando lo anterior no basta → programado, revisado y testeado.
>
> Así dos peluquerías pueden tener **flujos distintos** sin duplicar código, y añadir un negocio nuevo no toca la plataforma.

#### 5.1 Contrato de un pack

```python
# packs/salon/pack.py
from core.packs import Pack, Periodic
from .config import SalonConfig           # esquema de config (pydantic)
from . import handlers, blocks

pack = Pack(
    name="salon",
    version="1.0.0",
    title="Peluquería / estética",
    config_schema=SalonConfig,                       # valida la config de negocio
    default_recipe="recipes/default.yaml",           # receta si el tenant no tiene una propia
    blocks=blocks.ALL,                               # bloques propios del pack (además de los comunes)
    handlers={                                       # qué hace con cada tipo de evento
        "message": handlers.conversation,            # el motor de conversación (§5.3)
        "task:reminder": handlers.send_reminder,
        "task:review_request": handlers.send_review_request,
        "form": None,                                # este pack no recibe formularios
    },
    periodic=[Periodic("daily_summary", cron="0 8 * * *", handler=handlers.daily_summary)],
    connectors={"booking": "required", "llm": "optional", "stt": "optional"},
    whatsapp_templates="templates/whatsapp.yaml",    # plantillas que este pack necesita
    knowledge_template="knowledge/template.md",
    scenarios="scenarios/",                          # tests de conversación (§15.3)
)
```

**Descubrimiento:** al arrancar, la plataforma importa `packs/*/pack.py` y registra cada `pack`. Añadir un pack = añadir una carpeta. **No se toca la plataforma.**

**Tipos de handler:**

| Evento | Clave del handler | Ejemplo |
|---|---|---|
| Mensaje de un contacto | `message` | Conversación de reserva |
| Formulario recibido | `form` | Gimnasio: Google Form de un socio |
| Tarea programada que vence | `task:<nombre>` | Recordatorio de cita |
| Tarea periódica | en `periodic` | Resumen diario al dueño |
| Webhook externo | `webhook:<nombre>` | Aviso del software del cliente |
| Mensaje del dueño (bot del dueño) | `owner:<comando>` | `/bloquear jueves 15-17` |

#### 5.2 Config de negocio

Cada pack define su esquema con **pydantic**. La consola genera el formulario a partir de él y valida antes de guardar.

```python
# packs/salon/config.py
class Service(BaseModel):
    id: str
    name: str = Field(max_length=24)        # cabe en una fila de lista de WhatsApp
    duration_min: int
    price_eur: Decimal | None = None
    staff_ids: list[str] = []               # vacío = cualquiera

class SalonConfig(BaseModel):
    business_name: str
    timezone: str = "Europe/Madrid"
    services: list[Service]
    staff: list[Staff] = []
    opening_hours: WeeklyHours              # {mon: ["10:00-14:00","17:00-21:00"], ...}
    booking_horizon_days: int = 14
    min_notice_min: int = 60
    reminder_hours_before: int = 24
    waitlist_enabled: bool = False
    review_request_enabled: bool = False
    review_url: HttpUrl | None = None
    texts: SalonTexts = SalonTexts()        # todos los textos con valores por defecto, editables
```

- Se guarda en `tenant_pack_assignment.config` (JSON) con **historial de versiones** (`config_version`) y quién lo cambió (auditoría).
- **Efecto inmediato** (sin despliegue).
- Puede editarse desde la consola, el Django Admin o el bot del dueño (comandos concretos).

#### 5.3 Motor de conversación

> **En pocas palabras.** Una conversación es una sucesión de **preguntas y respuestas**. El motor sabe en qué bloque y en qué paso está cada conversación, pasa el mensaje al bloque actual y, cuando el bloque termina, pasa al siguiente de la receta. También se encarga de lo común a todas: "MENU", "HUMANO", "CANCELAR", preguntas libres a la IA y conversaciones abandonadas.

**El contexto que recibe un pack:**

```python
class PackContext:
    tenant: TenantInfo              # id, nombre, zona horaria, idioma
    config: BaseModel               # la config validada (p. ej. SalonConfig)
    recipe: Recipe                  # la receta efectiva del tenant
    contact: ContactInfo            # id, nombre conocido, canal
    conversation: ConversationView  # estado actual (lectura); se modifica vía resultado
    connectors: Connectors          # ctx.connectors.booking.get_slots(...)  (§7)
    knowledge: Knowledge            # documento + ayudante de FAQ
    clock: Clock                    # "ahora" inyectable (tests deterministas)
    channel: ChannelCapabilities    # qué soporta el canal (botones, listas, plantillas…)
    idem: IdempotencyHelper         # genera claves idempotentes (§10.4)
    log: Logger
```

**Lo que devuelve:**

```python
class HandleResult:
    replies: list[Reply]            # Text, Buttons, List, Template, Flow, Media, Location
    actions: list[Action]           # ScheduleTask, CancelTask, Handoff, NotifyOwner, EmitMetric
    state: ConversationState | None # nuevo estado (None = sin cambios)
```

**Un bloque:**

```python
class Block(Protocol):
    name: str                        # "choose_slot"
    version: int                     # se incrementa si cambia el formato de su estado
    Params: type[BaseModel]          # parámetros que acepta en la receta

    def start(self, ctx, params, data) -> Step: ...
    def on_input(self, ctx, params, data, block_state, msg) -> Step: ...

# Resultados posibles de un paso:
Ask(replies, expect=Expect.choice(options) | Expect.text() | Expect.date() …, block_state={...})
Done(result=...)        # el bloque terminó; su resultado se guarda en data[<id del paso>]
Back()                  # volver al paso anterior
Abort(reason)           # cancelar el recorrido → volver al menú
Handoff(reason)         # derivar a humano
```

**Cómo el motor ejecuta una receta:**

```
mensaje → ¿comando global? (MENU / HUMANO / CANCELAR / BAJA)        → lo resuelve el motor
        → ¿conversación en modo humano?                              → no responde; reenvía al dueño
        → ¿conversación caducada (inactiva > N min)?                 → reinicia en el recorrido de entrada
        → bloque actual.on_input(msg)
             ├─ Ask   → responder y guardar block_state
             ├─ Done  → guardar resultado → siguiente bloque.start()  (o fin del recorrido)
             └─ el bloque no entiende el mensaje (texto libre)
                   → ¿IA de FAQ activa? → responder con el knowledge y REPETIR la pregunta actual
                   → si no → mensaje de "no te he entendido" + repetir la pregunta
```

**Estado de una conversación** (JSON en `conversation.state`):

```json
{
  "recipe_version": "salon@default#a1b2c3",
  "journey": "book",
  "step_index": 2,
  "step_id": "day",
  "block": {"name": "choose_day", "version": 1, "state": {"page": 0}},
  "data": {"service": {"id": "corte", "duration_min": 30}},
  "started_at": "2026-10-03T10:12:00+02:00"
}
```

#### 5.4 Bloques

**Bloques comunes** (en `core/blocks/`, los usa cualquier pack):

| Bloque | Qué hace | Parámetros típicos |
|---|---|---|
| `menu` | Muestra opciones y salta al recorrido elegido | `options`, `text_key` |
| `choose_option` | Elegir de una lista (estática o de la config) | `source`, `text_key` |
| `ask_text` | Pedir un texto con validación | `validate` (`name`, `email`, `regex`), `skip_if_known` |
| `ask_date` | Pedir una fecha (botones o texto, con IA de extracción opcional) | `min`, `max`, `allow_free_text` |
| `confirm` | Resumen + Sí/No | `summary_template` |
| `show_info` | Enviar un texto o documento y terminar | `text_key`, `media` |
| `handoff` | Derivar a humano con motivo | `reason` |
| `faq` | Recorrido de pregunta libre con la IA | `max_turns` |

**Bloques de reservas** (en `core/blocks/booking/`, para peluquería, clínica, talleres…; usan el conector `booking`):

| Bloque | Qué hace |
|---|---|
| `choose_service` | Lista de servicios desde la config |
| `choose_staff` | Elegir profesional (o "me da igual") |
| `choose_day` | Días con hueco (consulta `booking.get_available_days`) |
| `choose_slot` | Horas libres del día (consulta `booking.get_slots`) |
| `confirm_booking` | Confirma y **crea la cita** de forma idempotente; programa el recordatorio |
| `list_my_bookings` | Citas futuras del contacto |
| `cancel_booking` | Elegir cita → confirmar → cancelar → cancelar recordatorio → avisar a la lista de espera |
| `reschedule_booking` | Cancelar + reservar, en un solo recorrido |
| `join_waitlist` | Apuntarse a la lista de espera de un día o servicio |

**Bloques propios de un pack** (p. ej. `packs/bus/blocks/ask_route.py`): igual contrato, solo visibles en ese pack.

**Regla de versionado de bloques [DECIDIDO]:**

- Solo existe **una versión de cada bloque** en ejecución (todo va en la misma imagen).
- Cambio **compatible** (mejora interna, parámetro nuevo opcional): se modifica el bloque. Los tests de escenarios de **todos los tenants** (§15.3) garantizan que nadie se rompe.
- Cambio **incompatible** (cambia el formato del estado o el comportamiento esperado): **bloque nuevo** (`choose_slot_v2`). Cada tenant se migra explícitamente en su receta. El viejo se borra cuando ninguna receta lo usa (CI lo detecta).
- Si una conversación en curso tiene un `block.version` que ya no existe → el motor **reinicia con amabilidad** ("Perdona, he tenido que reiniciar; ¿qué querías hacer?") y lo registra.

#### 5.5 Recetas

> **En pocas palabras.** La receta es la "lista de pasos" de un negocio. No tiene condiciones ni lógica: solo qué bloques, en qué orden y con qué parámetros. Por eso es imposible que se convierta en un lenguaje de programación difícil de mantener.

```yaml
# packs/salon/recipes/default.yaml  (receta por defecto del pack)
entry: main_menu
journeys:
  main_menu:
    - {id: menu, block: menu, params: {options: [book, my_bookings, cancel, info, human]}}
  book:
    - {id: service, block: choose_service}
    - {id: day,     block: choose_day,  params: {lookahead_days: "{{config.booking_horizon_days}}"}}
    - {id: slot,    block: choose_slot}
    - {id: name,    block: ask_text,    params: {validate: name, skip_if_known: true}}
    - {id: confirm, block: confirm_booking}
  my_bookings: [{id: list, block: list_my_bookings}]
  cancel:      [{id: cancel, block: cancel_booking}]
  info:        [{id: faq, block: faq}]
  human:       [{id: handoff, block: handoff, params: {reason: user_request}}]
```

```yaml
# tenants/peluqueria-norte/recipe.yaml  (receta propia: pide profesional y señal)
extends: salon/default
journeys:
  book:
    - {id: service, block: choose_service}
    - {id: staff,   block: choose_staff,  params: {allow_any: true}}
    - {id: day,     block: choose_day}
    - {id: slot,    block: choose_slot}
    - {id: name,    block: ask_text, params: {validate: name, skip_if_known: true}}
    - {id: deposit, block: request_deposit, params: {amount_eur: 5}}   # bloque [FUTURO]
    - {id: confirm, block: confirm_booking}
```

- **Dónde viven [DECIDIDO]:** en el repositorio (`packs/<pack>/recipes/` y `tenants/<slug>/recipe.yaml`). Pasan por PR y tests, y el comando `sync_packs` las registra en la BD en cada despliegue (`recipe_version`, con checksum).
- **Validación** (en CI y en `sync_packs`): bloques existentes, parámetros válidos según `Params`, referencias `{{config.x}}` válidas según el esquema de config, el recorrido de entrada existe, todo recorrido es alcanzable desde el menú.
- **Un tenant sin receta propia** usa la del pack: dar de alta un cliente estándar **no requiere despliegue**.
- **[FUTURO]** Editor de recetas en la consola, con ejecución de los escenarios antes de activar.

#### 5.6 Variantes en código

Cuando un tenant necesita algo que ni la config ni la receta cubren:

1. **Primero**, ¿se puede resolver con un parámetro nuevo en un bloque existente? Si es razonable que otros lo usen → parámetro.
2. **Si no**, bloque nuevo **del pack** (`packs/salon/blocks/combo_rules.py`), usado desde la receta de ese tenant.
3. **Solo como último recurso**, módulo de variante `tenants/<slug>/variant.py` que sustituye un handler concreto.

**Regla de los tres:** si la misma variante aparece en tres tenants, se convierte en parámetro o bloque del pack.

#### 5.7 Versiones y conversaciones en curso

- Cada conversación guarda la `recipe_version` con la que empezó y **la termina con esa versión** (la receta antigua sigue en la BD).
- Las conversaciones inactivas más de `conversation_timeout_min` (por defecto 30 min; configurable por pack) se reinician en el siguiente mensaje con la receta activa.
- Los cambios de **config** se aplican al momento incluso a conversaciones en curso (un precio nuevo se ve en el siguiente mensaje).

#### 5.8 Packs iniciales

**`salon` (peluquería / estética / barbería)**

- Recorridos: menú, reservar, mis citas, cancelar, cambiar, información (FAQ con IA), humano.
- Tareas: `reminder` (24 h antes, plantilla de WhatsApp con botones Confirmar/Cancelar), `review_request` (2 h después, opcional), `waitlist_offer` (al cancelar), `reactivation` [FUTURO].
- Periódicas: `daily_summary` al dueño (8:00).
- Conectores: `booking` (obligatorio), `llm` y `stt` (opcionales).
- Comandos del dueño: `/hoy`, `/manana`, `/bloquear <día> <franja>`, `/vacaciones <desde> <hasta>`, `/pausar`, `/reanudar`.

**`bus` (horarios de autobús)**

- Datos propios del pack (tablas del pack, con `tenant_id` y RLS): `Stop`, `StopAlias`, `Route`, `Trip`, `StopTime`, `ServiceCalendar`, `CalendarException`, `Incident`.
- Importador (cola `heavy`): Excel → validación con informe de errores → modelo canónico (inspirado en GTFS). Se sube desde la consola.
- Recorridos: `consulta` (origen → destino → fecha → próximas salidas), con bloque `ask_stop` (búsqueda difusa + alias), e `incidencias` (avisos activos).
- **Sin IA para los horarios**: la IA solo puede ayudar a interpretar el texto ("mañana a Écija por la tarde"), y el resultado siempre se valida con datos.
- Comandos del operador: `/incidencia <texto>`, `/fin_incidencia`.
- [FUTURO] Exportación GTFS.

**`gym` (seguimiento de socios)**

- Evento `form` (Google Form → Apps Script → webhook firmado).
- Handler: valida → calcula o clasifica (reglas; la IA solo si aporta y **sin datos de salud**) → ficha para el entrenador (Telegram, email o fila en una hoja) → bienvenida al socio (opcional).
- Periódicas: resumen semanal al entrenador.

#### 5.9 Cómo se añade un pack nuevo (checklist)

1. `packs/<nombre>/pack.py` con el manifest.
2. `config.py` con el esquema y valores por defecto.
3. Receta por defecto (si es conversacional) y bloques propios si hacen falta.
4. Handlers de tareas, formularios o webhooks.
5. Plantillas de WhatsApp y plantilla de knowledge.
6. **Escenarios de test** (mínimo: camino feliz de cada recorrido + 3 caminos de error).
7. Entrada en `docs/packs/<nombre>.md` (qué hace, qué config pide, qué conectores necesita).
8. PR → CI → revisión → despliegue → alta del tenant desde la consola.

---

### 6. Canales

> **En pocas palabras.** Cada canal (WhatsApp, Telegram, web chat…) tiene un **adaptador** que traduce sus mensajes a un formato común y viceversa. Los packs nunca saben por qué canal hablan: piden "botones" y el adaptador los convierte en lo que el canal soporte (botones, lista o texto numerado).

#### 6.1 Contrato común

```python
@dataclass
class InternalMessage:                    # lo que reciben los packs
    id: str                               # id del proveedor (wamid, update_id…)
    tenant_id: UUID
    channel_id: UUID
    contact_external_id: str              # teléfono E.164, id de Telegram, id de sesión web
    type: Literal["text","button","list","media","location","form_reply","command","system"]
    text: str | None
    payload: str | None                   # id del botón o de la fila elegida
    media: MediaRef | None                # audio/imagen/documento ya subido a S3
    received_at: datetime
    raw_ref: UUID                         # InboundEvent original

class ChannelAdapter(Protocol):
    type: str
    capabilities: ChannelCapabilities     # max_buttons, max_list_rows, supports_templates, window_hours…
    def verify(self, request, channel) -> bool                     # firma
    def parse(self, payload, channel) -> list[ParsedItem]          # mensajes + statuses
    def render(self, replies, channel) -> list[OutboundPayload]    # Reply → formato del canal
    def send(self, outbound, channel) -> SendResult                # llamada a la API
```

**Degradación** (reutilizada del repo, `degradation.py`): lista → botones → texto numerado, según `capabilities`. Un usuario que contesta "2" a un texto numerado se traduce al `payload` de la opción 2.

**Plan de envío** (`plan_sends`), común a todos los canales:

1. **Fusionar** los textos consecutivos de un mismo turno en un solo mensaje.
2. Si un texto va seguido de botones o lista, **meter el texto en el cuerpo** del interactivo.
3. Aplicar los límites del canal (degradar o partir).
4. **Ventana de 24 h** (WhatsApp): si está cerrada y la respuesta no es una plantilla → sustituir por la plantilla configurada o, si no hay, no enviar y alertar.
5. Crear un `SendIntent` por mensaje final, en orden.

**Por qué la fusión es obligatoria:** desde el 1/10/2026 Meta cobra cada mensaje de servicio por encima de 1.000 al mes por número; menos mensajes = menos coste para el cliente.

#### 6.2 WhatsApp (Cloud API)

**Dos modos de conexión** (`whatsapp_connection.app_mode`):

| Modo | Cuándo | Quién es dueño de la Meta App | Qué guardamos |
|---|---|---|---|
| `client_app` | Ahora (sin alta ni verificación propia) | **El cliente** (en su Business Portfolio). Nos da acceso de desarrollador o un System User | `app_id`, `app_secret` (cifrado), `access_token` (cifrado), `waba_id`, `phone_number_id` |
| `platform_app` | Con alta + verificación + Tech Provider [FUTURO] | **Nosotros**. El cliente conecta con Embedded Signup | `waba_id`, `phone_number_id`, token de negocio (cifrado) |

El código es el mismo en los dos modos: lo único que cambia es **de dónde salen el secreto (para verificar la firma) y el token (para enviar)**.

**Entrada:**

- `GET /webhooks/whatsapp/<channel_id>`: verificación inicial (`hub.verify_token` por canal).
- `POST /webhooks/whatsapp/<channel_id>`:
  - verificar `X-Hub-Signature-256` (HMAC-SHA256 del cuerpo con el `app_secret` del canal), en tiempo constante;
  - comprobar que el `phone_number_id` del payload es el del canal;
  - separar en elementos: mensajes (`text`, `interactive.button_reply`, `interactive.list_reply`, `audio`, `image`, `document`, `location`, `nfm_reply` de Flows) y statuses (`sent`, `delivered`, `read`, `failed`, con su `pricing`);
  - un `InboundEvent` por elemento (`provider_event_id` = `wamid` o `wamid:status`).
- Los audios, imágenes y documentos se descargan (con el token) **en el worker**, no en el webhook, y se guardan en S3.

**Salida:**

- `POST https://graph.facebook.com/<versión>/<phone_number_id>/messages`. La versión de la Graph API va en configuración (`META_GRAPH_VERSION`), nunca en el código.
- Tipos: `text`, `interactive` (`button` ≤ 3, títulos ≤ 20 caracteres; `list` ≤ 10 filas, títulos ≤ 24), `template`, `interactive` de tipo `flow`, `image`, `document`, `location`.
- Se guarda el `wamid` devuelto en `SendIntent` y en `Message`.

**Plantillas** (`message_template`): por tenant y canal, con nombre, idioma, categoría (`utility`/`marketing`/`authentication`), componentes y estado (`pending`/`approved`/`rejected`). Los packs declaran las que necesitan (`templates/whatsapp.yaml`); el alta del tenant las crea y las envía a aprobación vía API; la consola muestra su estado.

**Costes:** el status trae `pricing.category` y `pricing.billable` → `message.pricing_category`, `message.billable`. El coste se calcula con la tabla `channel_rate` (país, categoría, precio, vigente desde) y se acumula en `usage_daily`. **Alerta** cuando un número pasa de 800 mensajes de servicio en el mes (el límite gratuito es 1.000).

**WhatsApp Flows** [FUTURO cercano]: bloque `whatsapp_flow` que envía un Flow (p. ej. reserva en una sola pantalla) y recibe `nfm_reply`. En canales sin Flows se degrada a la secuencia de bloques normal.

#### 6.3 Telegram

- Un **bot por tenant** (creado con @BotFather; el token se guarda cifrado) para hablar con sus clientes, si el tenant usa Telegram como canal.
- Alta: `setWebhook(url=…/webhooks/telegram/<channel_id>, secret_token=<aleatorio por canal>, allowed_updates=[message, callback_query])`.
- Verificación: cabecera `X-Telegram-Bot-Api-Secret-Token` comparada en tiempo constante.
- `provider_event_id` = `update_id`.
- Botones: `InlineKeyboardMarkup` → `callback_query.data` = `payload`; se responde siempre con `answerCallbackQuery`. Las capabilities admiten muchos más botones que WhatsApp, sin ventana de 24 h y gratis.
- [FUTURO] Mini Apps para formularios ricos; Business Mode.

#### 6.4 Web chat

- Widget JS (`<script src="https://widget.<dominio>/w.js" data-channel="<public_id>">`) servido desde S3/CloudFront [FUTURO para producción; en desarrollo se usa `chat_ui.html`].
- `POST /webchat/<public_id>/messages` → `InboundEvent` (como cualquier canal).
- `GET /webchat/<public_id>/messages?after=<cursor>` con **long-polling** (espera hasta 25 s a que haya respuestas). Sin WebSockets.
- Identidad: cookie con un id de sesión aleatorio → `contact_external_id`.
- Protección: rate limit por IP y sesión, CORS solo para el dominio del tenant y tamaño máximo de mensaje.

#### 6.5 Bot del dueño (consola por Telegram)

> **En pocas palabras.** Un único bot de Telegram de la plataforma con el que cada dueño de negocio (y tú) recibe avisos y da órdenes. Es gratis, rápido de construir y evita tener que hacer una web para clientes durante mucho tiempo.

- **Un solo bot de plataforma** (token en los secretos de plataforma).
- **Vinculación:** la consola genera un enlace `https://t.me/<bot>?start=<código de un solo uso, caduca en 24 h>` → al abrirlo se crea `owner_link(tenant_id, telegram_user_id, role)`. Un usuario de Telegram puede estar vinculado a varios tenants (elige con `/negocio`).
- **Avisos** (acción `NotifyOwner`): derivación a humano, resumen diario, cancelaciones, errores de un conector del tenant, plantilla rechazada.
- **Derivación con respuesta:** el aviso de derivación incluye el hilo; el dueño **responde al mensaje en Telegram** → se envía al contacto por su canal original (dentro de la ventana de 24 h; fuera, se avisa de que hace falta una plantilla). Botón "Devolver al bot" → la conversación vuelve a modo bot.
- **Comandos:** los genéricos de plataforma (`/negocio`, `/pausar`, `/reanudar`, `/estado`, `/soporte <texto>`) y los del pack (`owner:<comando>`).
- **Bot de operación (ops):** el mismo mecanismo con rol `operator` para ti: alertas de la plataforma (§18.4).

#### 6.6 Canales futuros (contrato ya previsto)

| Canal | Notas de diseño |
|---|---|
| Instagram DM / Messenger | Mismo modelo de webhooks y firma de Meta → reutiliza el 70 % de WhatsApp; ventana de 24 h; requiere App Review |
| Email | Entrada con inbound parse (POST al webhook); salida con SES o Brevo vía Anymail |
| SMS | Categoría `notification`, adaptador Twilio; solo como respaldo |
| Voz | Plataforma externa (Retell/Vapi) que llama a **endpoints de herramientas** (`/tools/<tenant>/booking/slots`…) firmados; nuestro backend sigue siendo la fuente de verdad |

---

### 7. Conectores

> **En pocas palabras.** Un conector es la forma estándar de hablar con un sistema externo (Google Calendar, la IA, una hoja de cálculo…). Están agrupados por **categoría**: todos los conectores de "reservas" tienen los mismos métodos, sea Google Calendar o el software Flowww. Así un pack pide "dame los huecos del jueves" sin saber qué sistema hay detrás, y **varios packs usan el mismo conector**. Los reintentos, los tiempos máximos y el "fusible" (circuit breaker) se aplican en un solo sitio para todos.

#### 7.1 Piezas

```
Categoría (interfaz con métodos tipados)     core/connectors/interfaces/booking.py
   └─ Adaptador (implementación concreta)     platform/connectors/adapters/google_calendar/
        └─ Manifest (nombre, categoría, esquema de config, credenciales que necesita)
Binding (tenant + categoría → adaptador + config + credencial)   tabla connector_binding
Registro + políticas (reintentos, timeout, circuit breaker, métricas, logs)   platform/connectors/registry.py
```

**Uso desde un pack:**

```python
slots = ctx.connectors.booking.get_slots(day=date(2026,10,9), service_id="corte", staff_id=None)
```

`ctx.connectors.booking` es un **proxy** que: resuelve el binding del tenant → descifra la credencial en memoria → llama al adaptador con un timeout → reintenta si el error es transitorio → actualiza el circuit breaker → registra la duración, el resultado y el coste.

#### 7.2 Categorías iniciales y sus métodos

**`booking`** (reservas). Adaptadores: `google_calendar` (portado del repo), `mock`; [FUTURO] `flowww`, `booksy_partner`.

```python
class BookingConnector(Protocol):
    def get_available_days(self, service_id, staff_id, from_date, days) -> list[date]
    def get_slots(self, day, service_id, staff_id) -> list[Slot]          # Slot(start, end, staff_id)
    def create_booking(self, req: BookingRequest, idempotency_key: str) -> Booking
    def cancel_booking(self, booking_id, reason) -> None
    def get_booking(self, booking_id) -> Booking | None
    def list_bookings_for_contact(self, contact_ref, from_date) -> list[Booking]
    def list_bookings(self, start, end) -> list[Booking]                 # resumen del dueño
    def block_time(self, start, end, staff_id, reason) -> BlockRef        # /bloquear
```

Para Google Calendar: los servicios y horarios vienen de la config; `compute_slots()` (función pura del repo) calcula los huecos restando los eventos; el evento se crea con un **id determinista** derivado de la clave idempotente (Google rechaza el duplicado); los datos de la cita van en la descripción del evento (formato del parser actual).

**`llm`** (IA de texto). Adaptadores: `gemini` (portado), `mistral`, `anthropic`, `mock`.

```python
class LLMConnector(Protocol):
    def answer(self, question, knowledge, history, policy) -> Answer       # Answer(text, grounded, tokens_in, tokens_out)
    def classify(self, text, labels: list[str], instructions) -> Classification
    def extract(self, text, schema: type[BaseModel], instructions) -> BaseModel | None
```

**`stt`** (voz a texto): `transcribe(media_ref, language) -> Transcript`. Adaptadores: `openai_transcribe`, `deepgram`, `mock`.

**`sheets`**: `append_row(sheet_ref, values)`, `read_range(sheet_ref, range)`. Adaptador: `google_sheets`.

**`email`**: `send(to, subject, html, text, attachments)`. Adaptadores: `ses`, `brevo` (vía Anymail).

**`http`** (escape hatch): `request(name, params)`. La config define método, URL y cabeceras por nombre de operación; solo HTTPS y solo dominios declarados en la config.

**`notification`** [FUTURO]: `send_sms(...)`. Adaptador: `twilio`.

Los **servicios de plataforma** que no varían por tenant (almacenamiento S3, cifrado, reloj) no son conectores: son servicios internos.

#### 7.3 Manifest de un adaptador

```python
# platform/connectors/adapters/google_calendar/manifest.py
manifest = ConnectorManifest(
    name="google_calendar",
    category="booking",
    config_schema=GoogleCalendarConfig,      # calendar_id, timezone, slot_step_min, buffer_min…
    credentials=CredentialSpec(kind="google_service_account", fields=["json"]),
    factory=GoogleCalendarAdapter,
    policies=Policies(timeout_s=8, retries=3, backoff="exponential", breaker_threshold=5, breaker_cooldown_s=60),
    health_check="check_access",             # método usado por el health check del tenant
)
```

El esquema de config genera el **formulario en la consola** y valida el binding. Añadir un adaptador = añadir una carpeta con su manifest: la consola y el registro lo descubren solos.

#### 7.4 Errores y políticas

| Error del adaptador | Significado | Qué hace el registro |
|---|---|---|
| `TransientConnectorError` | Red, 5xx, 429, timeout | Reintenta con backoff; cuenta para el circuit breaker |
| `PermanentConnectorError` | 4xx, credencial revocada, dato inválido | No reintenta; avisa (dueño y ops si es credencial) |
| `ConnectorUnavailable` | Circuit breaker abierto | Falla rápido; el bloque muestra "ahora mismo no puedo consultar la agenda, inténtalo en unos minutos" o deriva a humano |

- **Circuit breaker** por (tenant, binding), en memoria de cada proceso (suficiente al principio). El estado se refleja en `integration_health` para la consola.
- **Idempotencia** de las operaciones que escriben (`create_booking`, `append_row`): la clave la genera `ctx.idem` (§10.4).

#### 7.5 Pruebas de conectores

| Nivel | Qué | Dónde |
|---|---|---|
| Unit | Lógica pura (`compute_slots`, parser) | CI |
| Contract | El adaptador contra **respuestas reales grabadas** (fixtures JSON de la API) | CI |
| Mock | `mock` de cada categoría, programable por escenario | Tests de packs |
| Smoke | Contra la API real con una cuenta de prueba | Manual en staging, antes de activar un adaptador nuevo |

---

### 9. Datos

> **En pocas palabras.** Toda la información vive en **una base de datos PostgreSQL**. Cada fila que pertenece a un cliente lleva su `tenant_id`, y la propia base de datos impide que un cliente vea datos de otro (Row-Level Security), aunque el código tenga un error.

#### 9.1 Modelo

Todas las tablas marcadas con 🔒 llevan `tenant_id` y RLS. Las claves primarias son UUID. Las fechas, `timestamptz`.

**Clientes y acceso**

```text
tenant 🔒*            id, slug, name, status[onboarding|active|paused|offboarded], timezone, locale,
                     plan, llm_monthly_token_quota, created_at
user                 (Django) — tú y, en el futuro, dueños con acceso al portal
tenant_membership 🔒  tenant_id, user_id, role[owner|staff|viewer]
owner_link 🔒         tenant_id, telegram_user_id, role[owner|staff|operator], linked_at
```
\* `tenant` tiene RLS sobre su propio `id`.

**Packs y configuración**

```text
pack_version         pack_name, version, checksum, registered_at            (plataforma, sin tenant)
recipe_version       id, pack_name, tenant_id NULL, name, checksum, content jsonb, registered_at
tenant_pack_assignment 🔒  tenant_id, pack_name, recipe_version_id, config jsonb, config_version,
                          enabled, updated_by, updated_at
config_history 🔒     tenant_id, config_version, config jsonb, changed_by, changed_at
knowledge_doc 🔒      tenant_id, version, content_md, is_active, created_by, created_at
```

**Canales**

```text
channel 🔒            id, tenant_id, type[whatsapp|telegram|webchat|…], status, public_id, display_name
whatsapp_connection 🔒  channel_id, tenant_id, app_mode[client_app|platform_app], app_id,
                       app_secret_cred, access_token_cred, verify_token, waba_id, phone_number_id,
                       display_phone_number, quality_rating, messaging_limit_tier
telegram_connection 🔒  channel_id, tenant_id, bot_username, bot_token_cred, webhook_secret
message_template 🔒   tenant_id, channel_id, name, language, category, status, components jsonb
channel_rate         country, channel_type, category, price_eur, valid_from       (plataforma)
```

**Conectores y credenciales**

```text
credential 🔒         id, tenant_id, kind, ciphertext, encrypted_data_key, kms_key_id, created_at, rotated_at
connector_binding 🔒  tenant_id, category, adapter, config jsonb, credential_id, enabled
integration_health 🔒 tenant_id, binding_id, last_ok_at, last_error_at, last_error, breaker_state
```

**Conversaciones**

```text
contact 🔒            id, tenant_id, channel_type, external_id, display_name, phone_e164,
                     opt_in_marketing, opt_in_at, blocked, created_at        UNIQUE(tenant_id, channel_type, external_id)
conversation 🔒       id, tenant_id, channel_id, contact_id, mode[bot|human], human_until,
                     state jsonb, recipe_version_id, last_inbound_at, last_activity_at, opened_at
message 🔒            id, tenant_id, conversation_id, direction[in|out], provider_message_id, type,
                     content jsonb, status, pricing_category, billable, cost_eur, created_at
handoff 🔒            id, tenant_id, conversation_id, reason, opened_at, resolved_at, resolved_by
```

**Procesamiento**

```text
inbound_event 🔒      id, tenant_id, channel_id, source[whatsapp|telegram|webchat|form|webhook|system],
                     provider_event_id, payload jsonb, received_at, status[pending|done|failed|ignored],
                     processed_at, error                        UNIQUE(channel_id, provider_event_id)
execution 🔒          id, tenant_id, inbound_event_id, pack, handler, recipe_version_id, status,
                     duration_ms, error_code, llm_tokens_in, llm_tokens_out, cost_eur, trace_id
send_intent 🔒        id, tenant_id, conversation_id, channel_id, seq, payload jsonb,
                     status[pending|sending|sent|failed|unknown], attempts, provider_message_id,
                     last_error, created_at, sent_at
side_effect 🔒        key, tenant_id, kind, status[started|done|failed], result jsonb, created_at   UNIQUE(tenant_id, key)
scheduled_task 🔒     id, tenant_id, key, task_name, run_at, payload jsonb,
                     status[scheduled|done|cancelled|failed], job_id, created_at   UNIQUE(tenant_id, key)
```

**Uso, auditoría y negocio**

```text
usage_daily 🔒        tenant_id, date, metric[messages_in|messages_out|service_billable|utility|marketing|
                     llm_tokens|stt_seconds|executions|errors], count, cost_eur
audit_event          id, tenant_id NULL, actor, action, resource_type, resource_id, at, metadata jsonb
service_contract 🔒   tenant_id, plan, setup_eur, monthly_eur, start_date, dpa_signed_at, notes
```

**Tablas de packs** (p. ej. `bus_stop`, `bus_trip`…): viven en la app Django del pack, **siempre con `tenant_id` y RLS**. Los packs tienen modelos propios solo cuando manejan datos de dominio (autobuses); la peluquería no tiene tablas propias (las citas están en el calendario del cliente).

#### 9.2 Aislamiento entre clientes (RLS)

**Roles de PostgreSQL:**

| Rol | Lo usan | `BYPASSRLS` | Permisos |
|---|---|---|---|
| `app_owner` | Migraciones (tarea `migrate`) | — (dueño de las tablas) | DDL |
| `app_runtime` | `web` (webhooks, web chat) y `worker` | **No** | SELECT/INSERT/UPDATE/DELETE |
| `app_operator` | Consola y Django Admin (solo usuarios operador) | **Sí** | SELECT/INSERT/UPDATE/DELETE |

**Política tipo** (se crea en la migración de cada tabla 🔒):

```sql
ALTER TABLE conversation ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON conversation
  USING      (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
```

**Cómo se fija el tenant:**

- Worker: el decorador `@tenant_task` abre una transacción y ejecuta `SET LOCAL app.tenant_id = '<uuid>'` antes de la lógica.
- Webhooks: la vista resuelve el canal (`channel_id` de la URL) con una función SQL `SECURITY DEFINER` que solo devuelve `(tenant_id, tipo, estado)` del canal, y después fija `app.tenant_id`.
- **Sin tenant fijado no se ve ninguna fila** (`current_setting(..., true)` devuelve NULL): si alguien olvida fijarlo, falla de forma segura.
- Django usa dos alias de BD: `default` (`app_runtime`) y `operator` (`app_operator`), y un *database router* que usa `operator` solo en las vistas de consola y admin protegidas por permisos de operador.

**Integridad entre tablas:** las tablas padre tienen `UNIQUE(tenant_id, id)` y las hijas usan **claves foráneas compuestas** `(tenant_id, x_id)`. Así es imposible que un mensaje del tenant A apunte a una conversación del tenant B.

**Tests de aislamiento (bloquean el merge):** con tenants A y B, para cada tabla 🔒: A no puede SELECT/UPDATE/DELETE filas de B; no puede insertar con `tenant_id` de B; no puede crear una FK que apunte a B; sin tenant fijado se leen 0 filas; un job de A no ve datos de B. **Un test genérico recorre todas las tablas 🔒 automáticamente**: si alguien añade una tabla con `tenant_id` y olvida la política, el test falla.

#### 9.3 Retención [DECIDIDO inicialmente; se ajusta en el DPA]

| Dato | Retención | Cómo |
|---|---|---|
| `inbound_event.payload` | 30 días | Tarea nocturna: se vacía el payload y se conservan los metadatos |
| `message.content` | 12 meses | Después, se anonimiza (sin texto, se conservan tipo y fechas) |
| Ficheros en S3 (audios, documentos) | 90 días (salvo packs que los necesiten, p. ej. gestoría) | Regla de ciclo de vida de S3 |
| `execution`, `send_intent` | 6 meses | Borrado |
| `usage_daily` | Indefinido (agregado, sin datos personales) | — |
| `audit_event` | 3 años | Borrado |
| Tenant dado de baja | Exportación + borrado en ≤ 30 días | Proceso de baja (§19.2) |

#### 9.4 Migraciones de esquema

- Siempre **compatibles con la versión anterior del código** (patrón expand/contract):
  1. *Expand*: añadir la columna o tabla nueva (nullable o con valor por defecto).
  2. Desplegar el código que usa lo nuevo.
  3. *Contract*: en un despliegue posterior, quitar lo viejo.
- Prohibido en un solo paso: renombrar o borrar columnas en uso, `NOT NULL` sin valor por defecto en tablas grandes, cambios de tipo que bloqueen la tabla.
- CI: `makemigrations --check` (no faltan migraciones) + `django-migration-linter` / `squawk` (detecta operaciones peligrosas).
- **Las migraciones que tocan RLS, roles o borran datos requieren revisión humana explícita** (etiqueta `needs-human-db-review` en el PR).

---

### Non-functional requirements

| Clave | Requisito | Implicación arquitectónica |
|---|---|---|
| `nfr.scale` | ~50 negocios, ~75.000 mensajes/mes, picos de 1–2 mensajes/segundo | Un worker con concurrencia 8 da margen de 10 a 50 veces sobre la carga (§20) |
| `nfr.availability` | Pueden caer unos minutos ocasionalmente; no se promete 24/7 | Despliegues sin corte y alarmas (§16.3, §18.4); Multi-AZ solo al tener 3–5 clientes de pago (§17.7) |
| `nfr.latency` | Responder en pocos segundos | El webhook guarda, contesta 200 y encola; el envío va por la cola `outbound` (§1.2, §10.1) |
| `nfr.data_sensitivity` | Datos personales (teléfonos, nombres, citas); sin pagos por ahora; datos de salud solo en packs futuros | Cifrado y RLS obligatorios (§9.2, §11); datos de salud fuera del MVP (ver Security boundaries) |
| `nfr.deployment` | AWS ECS Fargate en eu-south-2 (plan B eu-west-1), CDK en Python, dos cuentas (staging y prod) | Infraestructura como código (§17.5); cuentas separadas (§17.1); región pendiente de verificar (§17.2, §23) |

---

### 24. Glosario

| Término | Qué es (en llano) |
|---|---|
| **ALB** (Application Load Balancer) | La "puerta de entrada" de AWS: recibe las peticiones HTTPS y las reparte entre las copias de la web |
| **API** | La forma en que un programa le pide cosas a otro (p. ej. "envía este mensaje" a Meta) |
| **Autoescalado** | AWS añade o quita copias del programa según la carga |
| **Backoff exponencial** | Reintentar esperando cada vez más (2 s, 4 s, 8 s…) para no saturar |
| **CDK** | Herramienta para describir la infraestructura de AWS como código Python |
| **CI/CD** | Integración continua: tests automáticos en cada cambio. Despliegue continuo: publicar automáticamente |
| **Circuit breaker** (fusible) | Si un servicio externo falla mucho, se deja de llamarlo un rato para no empeorar las cosas |
| **CloudWatch** | El sistema de logs, métricas y alarmas de AWS |
| **Contenedor / imagen** | La imagen es el programa empaquetado con todo lo que necesita; un contenedor es esa imagen ejecutándose |
| **DSL** | Lenguaje propio para describir algo (el YAML de flujos actual lo es); lo evitamos |
| **ECR** | Almacén de imágenes de contenedor de AWS |
| **ECS / Fargate** | El servicio de AWS que ejecuta contenedores; con Fargate no hay servidores que gestionar |
| **Expand/contract** | Cambiar la BD en dos pasos para que la versión vieja y la nueva del código funcionen a la vez |
| **HMAC / firma** | Un código que demuestra que un mensaje viene de quien dice (Meta) y no ha sido modificado |
| **HTMX** | Librería que permite páginas web interactivas sin escribir JavaScript |
| **Idempotente** | Que hacerlo dos veces tiene el mismo efecto que hacerlo una |
| **KMS** | Servicio de AWS que guarda claves de cifrado; las claves nunca salen de él |
| **LLM** | Modelo de lenguaje (la IA de texto: Gemini, Claude, Mistral…) |
| **Long-polling** | El navegador pregunta "¿hay respuesta?" y el servidor espera hasta tenerla antes de contestar |
| **Migración** | Script que cambia la estructura de la base de datos |
| **Monolito modular** | Un solo programa organizado en módulos bien separados (lo contrario de microservicios) |
| **Multi-AZ** | Base de datos con una copia en otra zona de disponibilidad; si una falla, la otra toma el relevo |
| **OIDC** | Forma de que GitHub Actions entre en AWS sin guardar contraseñas |
| **PITR** | Restaurar la base de datos a un momento exacto del pasado (p. ej. a las 10:32 de ayer) |
| **Procrastinate** | Librería que usa Postgres como cola de trabajos pendientes |
| **pydantic** | Librería para definir y validar estructuras de datos en Python |
| **RDS** | Base de datos gestionada por AWS (copias, parches, réplica) |
| **RLS** (Row-Level Security) | Regla de la base de datos que solo deja ver las filas del cliente activo |
| **Rolling deploy** | Desplegar sustituyendo las copias de una en una, sin corte |
| **RPO / RTO** | Cuántos datos podrías perder como máximo / cuánto tardas en recuperar el servicio |
| **S3** | Almacenamiento de ficheros de AWS |
| **Security group** | Cortafuegos de AWS: quién puede conectarse a qué |
| **SSM Parameter Store** | Almacén de configuración y secretos de AWS |
| **Tenant** | Un cliente (negocio) de la plataforma |
| **Webhook** | Aviso que un servicio externo (Meta, Telegram) envía a nuestra URL cuando pasa algo |
| **Worker** | El proceso que hace el trabajo de fondo (lo que no es atender una petición web) |

---

## Testing Strategy

### 15. Calidad y revisión

> **En pocas palabras.** Ningún cambio llega a producción sin pasar por tests automáticos y por tu revisión. Los agentes de IA pueden programar, pero lo que toca datos, seguridad o producción siempre lo aprueba una persona. Cada negocio tiene "guiones de conversación" que se prueban automáticamente: si un cambio rompe la reserva de una peluquería concreta, CI lo detecta.

#### 15.1 Tipos de test

| Tipo | Qué prueba | Necesita | Ejemplo |
|---|---|---|---|
| **Unit** | Funciones puras, bloques, plan de envío | Nada | `compute_slots`, fusión de mensajes |
| **Scenarios** | Conversaciones completas de un pack, como caja negra | Nada (conectores mock) | "Reservar corte el jueves a las 17:00" |
| **Tenant scenarios** | Los mismos escenarios con la **receta y la config de cada tenant** | Snapshots de config (§15.3) | La reserva de Peluquería Norte con profesional |
| **Integration** | Webhook → cola → worker → envío, con BD real | Postgres | Evento duplicado no se procesa dos veces |
| **Isolation** | RLS en todas las tablas 🔒 | Postgres | §9.2 |
| **Contract** | Adaptadores contra respuestas grabadas | Fixtures | Parser de webhooks de Meta, respuestas de Calendar |
| **Migrations** | No faltan migraciones; no hay operaciones peligrosas | Postgres | — |
| **Evals IA** | Calidad de respuestas FAQ | Modelo real (manual o nocturno) | §8.3 |
| **Smoke** | Tras el despliegue: `/health` + conversación sintética por Telegram en staging | Entorno desplegado | — |

#### 15.2 Formato de escenario (se mantiene el del repo, ampliado)

```yaml
# packs/salon/scenarios/book_happy_path.yaml
name: Reserva completa de un corte
fixtures:
  now: "2026-10-05T10:00:00+02:00"
  booking:
    available_days: ["2026-10-08", "2026-10-09"]
    slots: {"2026-10-09": ["17:00", "17:30"]}
steps:
  - user: "hola"
    expect: {journey: main_menu, reply_contains: "¿Qué quieres hacer?", options_include: [book]}
  - user: {payload: book}
    expect: {step: service}
  - user: {payload: corte}
    expect: {step: day}
  - user: {payload: "day_2026-10-09"}
    expect: {step: slot, options_include: ["17:00"]}
  - user: {payload: "slot_17:00"}
    expect: {step: name}
  - user: "Lucía"
    expect: {step: confirm}
  - user: {payload: "yes"}
    expect:
      reply_contains: "¡Reserva confirmada!"
      connector_calls: [{booking.create_booking: {service_id: corte}}]
      scheduled: [{task: reminder, run_at: "2026-10-08T17:00:00+02:00"}]
```

#### 15.3 Escenarios por tenant (lo que permite flujos distintos sin miedo)

- Los ficheros `tenants/<slug>/recipe.yaml` (si existen) y un **snapshot de su config** (`tenants/<slug>/config.snapshot.json`, sin secretos ni datos personales) viven en el repositorio.
- El snapshot se actualiza con `manage.py export_config_snapshots`, que se ejecuta en prod y abre un PR automático cuando la config cambia (diario).
- CI ejecuta los escenarios del pack **para cada tenant** con su receta y su config. Si un tenant necesita escenarios propios (porque su receta lo cambia todo), los tiene en `tenants/<slug>/scenarios/`.
- Además, el "Comprobar salud" de la consola ejecuta lo mismo en prod con la config viva (§13.3).

#### 15.4 Puertas de CI (bloquean el merge)

1. `ruff` (estilo y errores) + `mypy` (tipos).
2. `import-linter` (regla de dependencias §4.2).
3. Tests unit + scenarios + tenant scenarios (sin BD; rápidos).
4. Tests integration + isolation + contract (con Postgres de servicio en CI).
5. `makemigrations --check` + linter de migraciones.
6. Validación de recetas y manifests (`manage.py validate_packs`).
7. `pip-audit` (dependencias con vulnerabilidades conocidas).
8. Construcción de la imagen Docker (que compile).

#### 15.5 Flujo de trabajo

```
Issue (qué y por qué)
  → rama corta desde main (feature/xxx)
  → agentes: planner (plan) → coder (código + tests) → reviewer (revisión contra el plan)
  → PR con descripción + checklist
  → CI en verde
  → REVISIÓN HUMANA (tú) — obligatoria en todos los PR
  → merge (squash) a main → despliegue automático a staging → smoke
  → despliegue a prod (manual, con aprobación)
```

**Siempre requieren tu aprobación explícita, aunque CI esté en verde:** migraciones que tocan RLS, roles o borran datos; cambios en `credentials/`, KMS o IAM; cambios en la verificación de firmas; infraestructura (CDK) de prod; cualquier acción sobre datos de prod; activar canales reales de un cliente.

Protección de rama: `main` sin push directo; PR con CI en verde y 1 aprobación.

#### 15.6 Revisión de cambios que no son código

- **Config de negocio y knowledge** (en la consola): validación por esquema al guardar + historial con diff + auditoría. Botón "probar" que ejecuta los escenarios con la nueva config antes de guardar.
- **Recetas**: son ficheros del repo → PR normal.
- **Plantillas de WhatsApp**: las revisa Meta; la consola muestra el estado.

---

## Documentation Plan


**Tipo de proyecto:** `mixed` (servicio web en Django, packs de negocio, conectores e infraestructura).

**Documentos que se mantienen:**

- `README.md`: qué es, estado del proyecto y cómo empezar.
- `docs/adr/`: una página por decisión importante (ADR-001 a ADR-018, enlazadas desde §0).
- `docs/packs/<pack>.md`: un documento por pack (qué hace, qué configuración pide, qué conectores necesita).
- `docs/runbooks/`: procedimientos de operación (§19.4).
- `docs/migracion.md`: cómo pasar del código actual a este diseño (§22).

**Documento principal de API:** `docs/api.md` (referencia de endpoints y webhooks).
