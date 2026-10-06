# design.md — Diseño técnico de la plataforma

> **Qué es este documento.** El diseño completo de la parte técnica: qué piezas hay, cómo se hablan, dónde vive cada cosa, cómo se programa un negocio nuevo, cómo se prueba, cómo se despliega y en qué infraestructura corre. El objetivo es que, con este documento delante, **solo queden dudas de implementación** (cómo escribir tal función), no de diseño (dónde va, quién la llama, cómo se despliega).
>
> **Para quién.** Para ti (Mariano) y para los agentes de programación. Cada sección empieza con un bloque **"En pocas palabras"** en lenguaje llano y después entra en el detalle técnico. Los términos técnicos están en el [glosario](docs/glosario.md).
>
> **Punto de partida.** Partimos de cero: no hay código previo que migrar.
>
> **Relación con otros documentos.**
> - `IDEA.md`: el problema y para quién es.
> - `docs/adr/`: una página por decisión de arquitectura (ADR-001 a ADR-005, enlazadas desde §0).
> - `CLAUDE.md` (sección "Reglas de FlowPilot"): las reglas de este documento resumidas para los agentes.
> - `docs/glosario.md`: los términos técnicos en lenguaje llano.
>
> **Cómo lo leen los agentes.** dev-team entrega este documento troceado por encabezado de nivel 2: lo que está antes de `## Architecture` no llega a ningún agente, y el planner recibe `## Module Contracts` + `## Testing Strategy`. Por eso los contratos que se programan (runtime, fiabilidad, packs, IA, canales, conectores, datos) viven en Module Contracts.
>
> **Convenciones.** **[DECIDIDO]**: no se discute salvo evidencia nueva. **[VERIFICAR]**: dato externo que hay que confirmar antes de usarlo. **[FUTURO]**: no se construye todavía; se indica brevemente cómo encajará.

---

## Índice

**Architecture**

0. [Resumen de decisiones](#0-resumen-de-decisiones)
1. [Conceptos clave](#1-conceptos-clave)
2. [Visión general](#2-visión-general)
3. [Principios y lo que no construimos](#3-principios-y-lo-que-no-construimos)
4. [Arquitectura de la aplicación: procesos y módulos](#4-arquitectura-de-la-aplicación-procesos-y-módulos)
5. [Seguridad y secretos](#5-seguridad-y-secretos)
6. [Security boundaries](#6-security-boundaries)
7. [Configuración: qué vive dónde](#7-configuración-qué-vive-dónde)
8. [Consola y Django Admin](#8-consola-y-django-admin)
9. [Entornos y desarrollo local](#9-entornos-y-desarrollo-local)
10. [Construcción y despliegue](#10-construcción-y-despliegue)
11. [Infraestructura en AWS](#11-infraestructura-en-aws)
12. [Observabilidad y alertas](#12-observabilidad-y-alertas)
13. [Operación técnica](#13-operación-técnica)
14. [Escalado](#14-escalado)
15. [Estructura del repositorio](#15-estructura-del-repositorio)
16. [Pendiente de verificar y decisiones abiertas](#16-pendiente-de-verificar-y-decisiones-abiertas)

**Module Contracts**

17. [Runtime de eventos (el corazón)](#17-runtime-de-eventos-el-corazón)
18. [Fiabilidad: que no se pierda ni se duplique nada](#18-fiabilidad-que-no-se-pierda-ni-se-duplique-nada)
19. [Packs: cómo conviven los distintos negocios](#19-packs-cómo-conviven-los-distintos-negocios)
20. [Inteligencia artificial](#20-inteligencia-artificial)
21. [Canales](#21-canales)
22. [Conectores](#22-conectores)
23. [Datos](#23-datos)
24. [Non-functional requirements](#24-non-functional-requirements)

**Testing Strategy**

25. [Calidad y revisión](#25-calidad-y-revisión)

**[Documentation Plan](#documentation-plan)** · [Glosario](docs/glosario.md)

---

## Architecture

### 0. Resumen de decisiones

| # | Tema | Decisión | ADR |
|---|---|---|---|
| 1 | Lenguaje y framework | **Python 3.14 + Django 5.2 LTS** (soporte hasta abril de 2028) | [ADR-001](docs/adr/ADR-001-monolito-modular.md) |
| 2 | Forma de la aplicación | **Un solo proyecto (monolito modular)** con dos procesos: `web` y `worker`, misma imagen | [ADR-001](docs/adr/ADR-001-monolito-modular.md) |
| 3 | Clientes (multitenant) | **Una base de datos PostgreSQL 18 compartida**, cada fila con `tenant_id`, aislamiento con **Row-Level Security** | [ADR-002](docs/adr/ADR-002-multitenancy-rls.md) |
| 4 | Trabajo en segundo plano | **Procrastinate 3.x** (cola guardada en el propio Postgres) con 4 colas: `inbound`, `outbound`, `scheduled`, `heavy` | [ADR-001](docs/adr/ADR-001-monolito-modular.md) |
| 5 | Entrada de eventos | **Guardar primero, contestar 200, procesar después**; "al menos una vez" + idempotencia (nada se pierde ni se duplica) | [ADR-004](docs/adr/ADR-004-entrada-idempotente.md) |
| 6 | Lógica de negocio | **Packs en Python**: recorridos definidos en Python dentro del pack + **config por tenant**. Recetas por tenant [FUTURO] | [ADR-003](docs/adr/ADR-003-packs-packcontext.md) |
| 7 | Dónde vive cada cosa | Lógica en el repositorio (con revisión y tests); datos de negocio (config, knowledge, datasets) en la base de datos (cambio inmediato) | — |
| 8 | Conectores | Servicios compartidos **por categoría con métodos tipados**; reintentos, timeouts y circuit breaker centralizados | — |
| 9 | IA | **`LLMConnector` genérico sobre Amazon Bedrock** (perfil de inferencia EU). La IA solo apoya: nunca decide reservas, precios ni horarios | — |
| 10 | Canales | **Un adaptador por canal**. Ahora WhatsApp y Telegram; email [FUTURO]. El cliente es dueño de sus cuentas (número, Meta, bots) | — |
| 11 | Interfaz interna | **Django Admin + consola mínima** (Django + HTMX). Portal de cliente [FUTURO] | — |
| 12 | Avisos al dueño | **Independientes del canal**: acción `NotifyOwner` + `owner_channel` por tenant. Telegram primero (bot de plataforma) | — |
| 13 | Infraestructura | **AWS: ECS Fargate (ARM) + RDS PostgreSQL 18 + ALB + S3 + KMS**, región **eu-south-2 (España)** [VERIFICAR servicios], plan B eu-west-1 | [ADR-005](docs/adr/ADR-005-aws-ecs-cdk.md) |
| 14 | Infraestructura como código | **AWS CDK en Python** | [ADR-005](docs/adr/ADR-005-aws-ecs-cdk.md) |
| 15 | Cuentas de AWS | **Dos cuentas** (staging y prod) bajo AWS Organizations | — |
| 16 | Despliegue | GitHub Actions con OIDC → imagen en ECR → tarea `migrate` → despliegue progresivo con vuelta atrás automática. **Staging automático, prod con aprobación manual**. `main` **sin protección de rama** (dev-team escribe `tasks/*.md` en `main`); todo el código entra por PR | — |
| 17 | Secretos | Credenciales de clientes **cifradas con KMS** en la BD; secretos de plataforma en **SSM Parameter Store / Secrets Manager**; Bedrock y SES sin claves (rol IAM) | — |
| 18 | Calidad | Puertas de CI obligatorias + **escenarios de conversación por pack** + revisión humana en todo PR | — |

---

### 1. Conceptos clave

> **En pocas palabras.** El vocabulario que usa el resto del documento y el código. Si un nombre aparece en el código, significa exactamente esto.

| Concepto | Qué es | Ejemplo |
|---|---|---|
| **Tenant** (cliente) | Un negocio que usa la plataforma. Todo dato pertenece a un tenant | "Peluquería Sur" |
| **Canal** | Una vía de comunicación de un tenant con sus contactos | El número de WhatsApp de la Peluquería Sur; su bot de Telegram |
| **Canal de pruebas** | Canal `test` interno: el simulador de la consola, para conversar con el bot de cualquier tenant sin WhatsApp ni Telegram | Probar el flujo de reserva antes de activar un cliente; smoke test |
| **Contacto** | Una persona que escribe al tenant por un canal (nunca "cliente": cliente = tenant) | Lucía, +34 6xx… |
| **Conversación** | El hilo entre un contacto y un tenant por un canal, con su estado (en qué paso está) | "Reservando: ha elegido corte, falta el día" |
| **Disparador** | Lo que pone en marcha la lógica de un pack: un mensaje, un formulario enviado, un cambio en el calendario del tenant, una tarea programada o periódica, un botón con payload ([FUTURO] un webhook externo) | El dueño crea una cita en su Google Calendar |
| **Evento** | Un disparador concreto ya guardado (`inbound_event`, su id es `event_id`) y pendiente de procesar | El mensaje "quiero cita" de las 10:12 |
| **Evento de calendario** | Una cita o evento de Google Calendar (`gcal_event_id`), **no** un `inbound_event` | La cita de Lucía del jueves a las 17:00 |
| **Pack** | El "programa" para un tipo de negocio: su lógica, sus recorridos, su esquema de configuración y sus tests | `salon`, `bus`, `gym` |
| **Bloque** | Una pieza reutilizable de conversación o de lógica | "elegir servicio", "elegir hueco", "confirmar reserva" |
| **Recorrido** (journey) | Una tarea completa que hace el usuario, formada por bloques en orden; **definido en Python dentro del pack** | "Reservar", "Cancelar", "Consultar horario" |
| **Config de negocio** | Los datos del tenant que usa el pack, validados con un esquema | Servicios, precios, duraciones, horario, textos, opciones on/off |
| **Knowledge** | El documento con la información del negocio que usa la IA para preguntas libres | Dirección, parking, política de cancelación |
| **Dataset** | Un conjunto de datos estructurados de un tenant que declara un pack, con parser y validador estricto, subido y versionado desde la consola | Los horarios de una empresa de autobuses (`timetable`) |
| **Conector** | Un servicio para hablar con un sistema externo, por categoría | `booking` (Google Calendar), `llm` (Bedrock), `email` (SES) |
| **Binding** | Qué conector concreto usa un tenant para una categoría, con su config y credenciales | Peluquería Sur: `booking` → Google Calendar, calendario X |
| **Respuesta** (reply) | Lo que el pack quiere decir al contacto | Texto, botones, lista, plantilla |
| **Acción** | Lo que el pack quiere que ocurra además de responder | Avisar al dueño, derivar a humano, programar una tarea |
| **Tarea programada** | Una acción para el futuro (`scheduled_task`), con clave única | `followup:conv_123` a las 10:00 de mañana |
| **Trabajo** / **tarea ECS** | *Trabajo*: un job de Procrastinate en una cola. *Tarea ECS*: un contenedor de Fargate (`web`, `worker`, `migrate`, `smoke`). No confundir con *tarea programada* | `process_event` · la tarea ECS `migrate` |
| **Intención de envío** (send intent) | Un mensaje que hay que enviar, guardado antes de enviarlo | — |
| **Variante** [FUTURO] | Código Python específico de un tenant, cuando la config no basta | Reglas de combinación de servicios muy particulares |

---

### 2. Visión general

> **En pocas palabras.** La plataforma es **un programa** que corre en la nube de Amazon. Le llegan cosas que atender desde distintas **fuentes**: mensajes de WhatsApp o Telegram, formularios, cambios en un Google Calendar, tareas programadas. Averigua **de qué negocio son**, ejecuta **la lógica de ese tipo de negocio** (la del "pack": peluquería, autobuses, gimnasio…) con **la configuración de ese negocio concreto** (sus precios, horarios, textos) y responde o actúa, usando si hace falta **conectores** (Google Calendar, la IA, el email…). Todo queda registrado: qué entró, qué se hizo, qué salió y cuánto costó.

#### 2.1 Las piezas

```
                        ┌────────────────────────── AWS (cuenta prod) ──────────────────────────┐
  WhatsApp (Meta) ───┐  │                                                                        │
  Telegram ──────────┤  │   ALB (puerta de entrada HTTPS)                                        │
  Formularios ───────┼──┼──►  ├─ api.<dominio>      ─┐                                           │
  Google Calendar ───┤  │     └─ consola.<dominio>  ─┤  (consola: simulador = canal de pruebas)  │
   (avisos de watch) │  │                            ▼                                           │
  EventBridge ───────┘  │                    ┌──────────────┐        ┌────────────────────────┐  │
   (métrica de cola)    │                    │  web (Django)│───────►│ PostgreSQL 18 (RDS)    │  │
                        │                    │  N copias    │        │  · datos de todo       │  │
                        │                    └──────────────┘        │  · cola de trabajos    │  │
                        │                    ┌──────────────┐        └───────────┬────────────┘  │
                        │                    │ worker       │◄───────────────────┘               │
                        │                    │ N copias     │── conectores ──► Google, Bedrock,  │
                        │                    └──────────────┘                  SES, Meta…        │
                        │   S3 (ficheros) · KMS (claves) · CloudWatch (logs, métricas, alarmas)  │
                        └────────────────────────────────────────────────────────────────────────┘
```

| Pieza | Qué es | Qué hace |
|---|---|---|
| **web** | Django ejecutándose con gunicorn | Atiende **todo el HTTP entrante**: webhooks, avisos de Calendar, formularios públicos, consola (con el simulador), admin, métrica interna de colas y `/health`. **No procesa eventos**: los verifica, los guarda y los encola |
| **worker** | El mismo código, ejecutando el proceso de Procrastinate | Procesa la cola: ejecuta la lógica de los packs, llama a conectores, envía mensajes, lanza tareas periódicas y barridos |
| **PostgreSQL** | Base de datos gestionada por AWS (RDS) | Guarda todo, incluida la cola de trabajos |
| **ALB** | Balanceador de carga de AWS | Puerta de entrada HTTPS; reparte peticiones entre las copias de `web` |
| **S3** | Almacenamiento de ficheros | Exportaciones ([FUTURO] ficheros recibidos) |
| **KMS** | Servicio de claves | Cifra las credenciales de los clientes |
| **CloudWatch** | Logs y métricas de AWS | Registros, gráficas, alarmas |
| **EventBridge** | Programador de AWS | Llama cada minuto al endpoint interno que publica la métrica de la cola (§12.3) |

#### 2.2 La vida de un evento (de punta a punta)

Ejemplo: una persona (un contacto) escribe "quiero cita" al WhatsApp de la Peluquería Sur. El camino es el mismo para cualquier fuente: solo cambia la entrada (paso 1–2).

1. **Meta** envía un aviso (webhook) a `https://api.<dominio>/webhooks/whatsapp/<public_id-de-la-app>`.
2. **web** identifica la app de WhatsApp (por la URL) y el canal → el negocio (por el `phone_number_id`), comprueba la firma (que de verdad viene de Meta), **guarda el evento** en `inbound_event` (si ya existía porque Meta lo reenvió, no hace nada más), **encola** un trabajo `process_event` y contesta **200 OK**. Tarda menos de 100 ms.
3. **worker** coge el trabajo. Hay un candado por conversación: si el mismo contacto manda dos mensajes seguidos, se procesan en orden, nunca a la vez.
4. **worker** carga (en una transacción corta) la conversación, la config de la peluquería y su estado, y pasa el evento al **pack de peluquería**.
5. El pack ejecuta el bloque actual (p. ej. "elegir servicio") y devuelve **respuestas** ("¿Qué servicio quieres?" + lista) y **acciones** (avisar al dueño, derivar a humano…). Si necesita datos externos, llama a un **conector** (Google Calendar para ver huecos), **sin transacción abierta**.
6. **worker** guarda en otra transacción corta el nuevo estado, las respuestas como "intenciones de envío" (`send_intent`) y las acciones, y encola los envíos.
7. **worker** (cola `outbound`) envía cada mensaje al canal, guarda el id que devuelve y marca el envío como hecho.
8. Si el canal los tiene, más tarde llegan avisos de estado ("entregado", "leído", con la categoría de precio en WhatsApp): se actualiza el mensaje y su coste.

Si algo falla a mitad (se reinicia un worker, Google no responde), el trabajo se reintenta y, gracias a la idempotencia (§18), **no se duplica nada**.

**Qué cambia según la fuente** (todo lo demás es común):

| | WhatsApp | Telegram | Aviso de Google Calendar | Email [FUTURO] |
|---|---|---|---|---|
| Verificación | HMAC-SHA256 del cuerpo con el `app_secret` | Cabecera `X-Telegram-Bot-Api-Secret-Token` (el `secret_token` del webhook) | Cabecera `X-Goog-Channel-Token` | Firma del proveedor de entrada |
| Id para deduplicar | `wamid` (mensajes) · `<wamid>:<status>` (avisos de estado) | `update_id` (prefijado con el canal; fiable solo dentro de la ventana de reintentos, §21.3) | El aviso no se guarda; cada evento cambiado: `<calendar_id>:<event.id>:<event.updated>` | `Message-ID` |
| Avisos de estado | Sí (enviado, entregado, leído, fallido, con precio) | No | — | Entregado / rebote / queja vía SES |
| Ventana de 24 h | Sí | No | — | No |

Los formularios públicos (§21.5) y el canal de pruebas (§21.4) entran por `web` con su propia protección y siguen el mismo camino desde el paso 2.

---

### 3. Principios y lo que no construimos

> **En pocas palabras.** Reglas que guían todas las decisiones, y lista de cosas que **no** construimos de entrada (usamos algo existente) con la señal que haría replantearlo.

#### 3.1 Principios

1. **Simple antes que flexible.** Una base de datos, un proyecto, una imagen. Se añade complejidad solo con un problema real delante.
2. **Contratos pequeños y estables, interior simple.** Las interfaces entre piezas (pack ↔ plataforma, canal, conector) son pocas y cambian poco; lo de dentro es código directo. Añadir un pack o un cliente no debe obligar a refactorizar. La abstracción se añade cuando algo se repite, no antes.
3. **Lo genérico en la plataforma, lo específico en los packs.** La plataforma no sabe qué es una peluquería.
4. **Lógica en código, datos en configuración.** Programar en Python; los precios, horarios, textos y datasets se cambian sin programar.
5. **Nada se pierde y nada se duplica.** Todo evento se guarda antes de procesarse y todo efecto externo es idempotente.
6. **Determinista por defecto, IA solo de apoyo.** Horarios, disponibilidad y precios salen de datos, nunca de la IA.
7. **Todo cambio de lógica pasa por tests y revisión.** Los de datos se validan contra un esquema o un validador.
8. **Portable.** El código no depende de AWS directamente salvo detrás de interfaces: si mañana cambiamos de nube, se cambia infraestructura, no código.
9. **Medir desde el primer día**: mensajes, costes, tiempos y fallos por cliente.
10. **Diseñado para una persona**: lo que no se automatiza o no se puede operar fácilmente, no se hace.

#### 3.2 No lo construimos de entrada; usamos lo existente que encaje

| No construimos | En su lugar | Se revisa cuando… |
|---|---|---|
| Microservicios | Monolito modular con dos procesos | Un módulo necesite escalar o desplegarse de forma independiente y separar el servicio `worker` no baste (§14) |
| Kubernetes | ECS Fargate | ECS deje de bastar (no se espera) |
| Una base de datos o un contenedor por cliente | BD compartida con RLS | Un cliente lo exija por obligación legal o contractual |
| Lenguaje propio de flujos (DSL) o recetas YAML | Recorridos en Python dentro del pack + config | Varios tenants repitan variaciones que la config no cubre (regla de los tres, §19.6) |
| Editor visual de flujos | Código revisado | Alguien que no programa tenga que crear flujos |
| Software de reservas propio | La agenda que el negocio ya usa (Google Calendar) | Un cliente use otro software: se añade un adaptador (§22.2) |
| Inbox omnicanal propio | Bot del dueño en Telegram; si hace falta, Chatwoot | Un cliente tenga varias personas atendiendo conversaciones |
| CRM propio | — (conector a uno existente si un cliente lo pide) | Un cliente lo pida |
| Facturación propia | Software homologado | — |
| App móvil o portal de cliente | Bot del dueño + operador | El bot del dueño no baste |
| Disponibilidad 24/7 garantizada | Despliegues sin corte y alarmas | Un cliente dependa del servicio en horas críticas |
| Agentes de IA autónomos | IA de apoyo acotada (§20) | — |

---

### 4. Arquitectura de la aplicación: procesos y módulos

> **En pocas palabras.** Es un solo proyecto Django dividido en "módulos" (carpetas con una responsabilidad cada una). Se ejecuta de dos maneras: como **web** (atiende peticiones) y como **worker** (hace el trabajo de fondo). Hay una regla de quién puede usar a quién, para que la plataforma nunca dependa de un negocio concreto.

#### 4.1 Procesos

| Proceso | Comando | Copias | Responsabilidad | Lo que NO hace |
|---|---|---|---|---|
| `web` | `gunicorn config.wsgi --worker-class gthread --workers 2 --threads 4` | 1 → N (autoescala) | Todo el HTTP entrante: webhooks de WhatsApp y Telegram, bot del dueño (`/webhooks/owner_bot`), avisos de Google Calendar (`/webhooks/gcal/…`), formularios públicos (`/f/…`), consola, admin, simulador, `POST /internal/metrics/queues`, `/health` | **Llamar a sistemas externos** (conectores, APIs de Meta/Telegram/Google), ejecutar packs, enviar mensajes. Las acciones de consola que necesitan salir fuera (comprobar salud, crear watch, `setWebhook`, plantillas) encolan un trabajo `console_action` (cola `scheduled`) y la página sigue su estado con HTMX (el simulador usa el camino normal, §21.4). `web` no ejecuta handlers de packs; sí importa sus esquemas de config y validadores de datasets (sin E/S externa) |
| `worker` | `python manage.py procrastinate worker --queues=… --concurrency=8` | 1 → N (autoescala) | Procesar eventos, enviar mensajes, tareas programadas, periódicas y barridos | Atender HTTP |
| `migrate` | `python manage.py migrate && python manage.py sync_packs` | Tarea puntual en cada despliegue | Actualizar el esquema de la BD y registrar las versiones de los packs | — |

**[DECIDIDO]** No hay un proceso "scheduler" aparte: las tareas periódicas las lanza el propio worker (Procrastinate tiene tareas periódicas integradas y garantiza que solo una copia las lanza).

**Cómo escala cada proceso:**

- **web**: por CPU (60 %) y por peticiones por tarea en el ALB. Es barato: solo verifica, guarda y encola; una tarea aguanta con holgura los picos previstos (§24).
- **worker**: por **antigüedad del trabajo más antiguo** de la cola `inbound` (`queue_oldest_age_s`). Esa métrica **no la publica el worker** (si el worker está caído o saturado, no la publicaría): la publica `web` cuando EventBridge le llama cada minuto (§12.3).

**Conexiones a la BD** (sin RDS Proxy, porque Procrastinate usa `LISTEN/NOTIFY`): cada hilo mantiene su conexión (`CONN_MAX_AGE`).

| Proceso | Conexiones por tarea (máx.) |
|---|---|
| `web` | 2 workers × 4 hilos = 8 (+ alias `operator` solo en vistas de consola) → **~10** |
| `worker` | concurrencia 8 + 1 de escucha (`LISTEN`) + 1 de tareas periódicas → **~10** |

Cálculo con el máximo de autoescalado (4 web + 6 worker + 1 migrate) ≈ 11 × 10 = **~110 conexiones**. db.t4g.small (2 GB) admite ~200 por defecto: hay margen. En staging (db.t4g.micro, ~100) hay 1 tarea de cada. Si se sube el máximo de tareas, se rehace este cálculo antes (§14).

#### 4.2 Módulos (apps de Django) y regla de dependencias

```
src/
  core/            Python puro, SIN Django: tipos y eventos de dominio, contratos, motor de conversación, bloques base
  flowpilot/       Django: tenancy, canales, conectores, datos, cola, runtime de eventos, consola
     tenancy/        Tenant, contexto de tenant (tenant_tx), RLS, router de BD
     channels/       whatsapp/, telegram/, test/, owner_bot/ (adaptadores + webhooks)
     forms/          Form, páginas públicas /f/<public_id>
     calendars/      watches de Google Calendar, webhook gcal, sincronización incremental
     datasets/       DatasetVersion, subida y validación, caché en memoria
     connectors/     registro, políticas (reintentos, circuit breaker), adaptadores por categoría
     conversations/  Contact, Conversation, Message, Handoff
     events/         InboundEvent, runtime (evento → pack → acciones), SendIntent, SideEffect
     packs_registry/ descubrimiento de packs (importlib sobre packs/*/pack.py) y sync_packs
     scheduling/     ScheduledTask, tareas periódicas y barridos
     credentials/    Credential (cifrado KMS)
     usage/          métricas de uso y coste, cuotas
     audit/          AuditEvent
     ops/            /health, /internal/metrics/queues
     console/        Consola (vistas Django + HTMX)
  packs/           Los negocios: salon/, bus/, gym/ … (dependen de core)
  config/          settings, urls, wsgi
```

El paquete se llama `flowpilot` (no `platform`, que choca con el módulo `platform` de la librería estándar de Python).

**Regla de dependencias [DECIDIDO]** (la comprueba `import-linter` en CI):

```
packs/*            ──►  core   (todo core salvo core.testing, que solo se importa desde tests)
packs/*/storage.py ──►  core + Django   (única excepción dentro de un pack; ver abajo)
flowpilot          ──►  core
flowpilot          ──►  packs  SOLO a través del registro de packs (flowpilot/packs_registry, descubrimiento), nunca importando un pack concreto
core               ──►  nada del proyecto (ni Django, ni flowpilot, ni packs)
packs/A            ──X  packs/B   (un pack no importa otro; lo compartido sube a core/blocks)
```

**Por qué importa:** cualquiera (persona o agente) puede trabajar en el pack de autobuses sin riesgo de romper la peluquería ni la plataforma, y los tests de `core` y de los packs corren sin base de datos.

**Datos propios de un pack [DECIDIDO, hoy sin uso].** Si un pack necesita guardar datos de dominio que no caben en la config, un dataset (§19.5) ni un sistema externo:

- El pack declara un `Protocol` con lo que necesita (p. ej. `class MembershipStore(Protocol): def get(...)`, en `packs/<nombre>/ports.py`) y su lógica solo ve ese protocolo vía `ctx.store`.
- La implementación vive en `packs/<nombre>/storage.py`: modelos Django (siempre con `tenant_id` y RLS, §23.2) + un repositorio que cumple el protocolo. Sus migraciones, en `packs/<nombre>/migrations/`.
- `import-linter` permite Django en `storage.py` (y en sus `migrations/`) y en ningún otro módulo del pack. El manifest referencia la implementación como texto (`store="packs.x.storage:make_store"`), así `pack.py` no importa Django.
- Los tests de lógica usan un repositorio falso en memoria que cumple el mismo protocolo.

Ningún pack inicial lo necesita: `salon` usa el calendario del cliente, `bus` un dataset y `gym` no guarda nada.

---

### 5. Seguridad y secretos

> **En pocas palabras.** Las contraseñas y tokens de los clientes se guardan **cifrados** con una clave que custodia AWS (KMS) y que nunca sale de ahí. Todo lo que llega de fuera se comprueba (firma, token o protecciones de formulario). La consola tiene doble factor. Los logs nunca contienen tokens ni teléfonos completos.

#### 5.1 Credenciales de clientes (cifrado por sobre)

```
Guardar:  KMS.GenerateDataKey(clave maestra) → clave de datos (en claro + cifrada)
          cifrar el secreto con AES-256-GCM usando la clave en claro → ciphertext
          guardar ciphertext + clave de datos cifrada + kms_key_id;  olvidar la clave en claro
Usar:     KMS.Decrypt(clave de datos cifrada) → clave en claro → descifrar → usar en memoria → olvidar
```

- Interfaz `CredentialCipher` con dos implementaciones: `KmsCipher` (staging y prod) y `LocalCipher` (desarrollo, clave en `.env`).
- Caché en memoria de las claves de datos descifradas, máx. 5 min (menos llamadas a KMS).
- Cada `Decrypt` queda registrado en CloudTrail (auditoría de quién accede a secretos).
- Qué se guarda como `credential`: tokens de WhatsApp y `app_secret`, tokens de bots de Telegram y **el JSON de la cuenta de servicio de Google**.
- Lo que solo hay que **comparar** (no usar) se guarda como **hash** y se compara en tiempo constante: `verify_token` de WhatsApp, `secret_token` de Telegram, token del watch de Calendar. Se muestra en claro una sola vez, al configurarlo.
- Rotación: la rotación anual de la clave maestra **se activa explícitamente en CDK** (`enable_key_rotation=True`; en claves propias no viene activada). Los tokens de los clientes se cambian desde la consola (`rotated_at`).

#### 5.2 Secretos de plataforma

En **SSM Parameter Store (SecureString)**, o Secrets Manager si necesitan rotación automática. Se inyectan como variables de entorno en las tareas ECS mediante la definición de tarea (nunca en la imagen ni en Git): `DJANGO_SECRET_KEY`, `DATABASE_URL_*`, `OWNER_BOT_TOKEN`, `OWNER_BOT_WEBHOOK_SECRET`, `INTERNAL_METRICS_SECRET`, `SENTRY_DSN`.

**Bedrock y SES no usan claves:** se autentican con el rol IAM de la tarea, con permisos mínimos (invocar solo los perfiles de inferencia permitidos; enviar solo desde la identidad verificada).

#### 5.3 Superficie de ataque

| Punto | Protección |
|---|---|
| Webhooks de canales y del bot del dueño | Firma (HMAC de Meta / `secret_token` de Telegram / `OWNER_BOT_WEBHOOK_SECRET`) **antes de nada**, en tiempo constante; tamaño máximo del cuerpo; URLs con `public_id` (nunca ids internos; única excepción: `/webhooks/gcal/<watch_id>`, con `watch_id = calendar_watch.id`, un UUIDv7 no adivinable y protegido además por el token); rate limit **solo de las peticiones con firma inválida** (Meta envía desde IPs compartidas) |
| Avisos de Google Calendar (`/webhooks/gcal/<watch_id>`) | `X-Goog-Channel-Token` comparado en tiempo constante con el del watch; el aviso no trae datos (solo dispara una sincronización); rate limit por watch |
| Formulario público (`/f/<public_id>`) | CSRF de Django; campo trampa (honeypot); rate limit por IP; tamaño máximo; sin subida de ficheros; casilla de consentimiento RGPD obligatoria; validación por esquema |
| Rate limit (en general) | Contadores en la caché de Django en BD (`DatabaseCache`) o, como aproximación, en memoria de cada tarea. La IP del cliente sale de `X-Forwarded-For` tomando la entrada que añade el ALB (último salto), nunca la primera |
| `POST /internal/metrics/queues` | Secreto en cabecera (`INTERNAL_METRICS_SECRET`) en tiempo constante; no devuelve datos (204) |
| Consola, admin y simulador | Solo en `consola.<dominio>`; **regla del ALB con lista de IPs permitidas** (casa, móvil vía VPN o Tailscale); login de Django + **2FA (django-otp)**; sesiones cortas; cada acción en `audit_event` |
| Red | RDS en subred privada (solo accesible desde las tareas); tareas con security group que solo acepta tráfico del ALB; ALB solo en 443 (80 redirige) |
| Dependencias | `uv.lock`; Dependabot; `pip-audit` en CI; imagen base mínima, actualizada mensualmente |
| Logs | Filtro que enmascara tokens, `Authorization` y teléfonos (últimos 4 dígitos) |
| Acceso a AWS | Sin usuarios IAM con claves; acceso por IAM Identity Center + MFA; CI por OIDC; roles de tarea con permisos mínimos (S3 de su bucket, KMS de su clave, SSM de su ruta, Bedrock y SES acotados) |
| Contenedores | Usuario no root, sistema de ficheros de solo lectura (salvo `/tmp`) |

#### 5.4 Datos personales

- Se registran en `message` solo los campos necesarios. Los datos de salud **nunca** en el texto libre de los packs, en formularios ni en prompts.
- Exportación y borrado por tenant y por contacto (derechos RGPD): comandos `export_tenant`, `forget_contact`.
- Los formularios piden consentimiento explícito y solo los campos que declara la config del pack.

---

### 6. Security boundaries

> **En pocas palabras.** Qué datos hay, por dónde entran y salen, y qué protege cada frontera.

**Clasificación de datos**

- **Datos personales** (teléfonos, nombres, emails, citas, contenido de mensajes y formularios): se minimizan (§20.2, §5.4), se cifran en reposo y se borran o anonimizan según §23.3.
- **Credenciales** (tokens de Meta y Telegram, JSON de la cuenta de servicio de Google, `app_secret`): cifradas con KMS en la base de datos (§5.1) o en SSM (§5.2). Nunca en claro.
- **Datos de salud**: fuera del alcance. No entran en prompts, formularios ni packs (§20.2, §5.4).
- **Uso y coste** (`usage_daily`): agregados, sin datos personales (§23.1).

**Qué cruza cada frontera**

Flujo: fuentes → web → cola → worker → conectores externos.

| Frontera | Qué cruza | Protección |
|---|---|---|
| Internet → webhooks de canales (`web`) | Payload del proveedor (mensajes y avisos de estado) | Firma verificada antes de guardar nada (§18.1, §5.3) |
| Google Calendar → `/webhooks/gcal/<watch_id>` | Aviso sin datos (cabeceras del canal de notificación) | Token del watch en tiempo constante; no guarda nada: solo encola una sincronización (§5.3, §22.2) |
| Formulario público → `web` | Datos que escribe una persona (nombre, email, teléfono…) | CSRF, honeypot, rate limit, tamaño máximo, consentimiento, esquema (§5.3, §21.5) |
| `web` → cola (Postgres) | `InboundEvent` con su payload; los trabajos reciben solo ids y candados sin teléfonos (§18.1, §18.2) | Idempotencia por `(tenant_id, source, provider_event_id)` (§18.1) |
| Cola → worker | Ids; el worker carga los datos del tenant | `tenant_tx` con `SET LOCAL app.tenant_id` y RLS (§17, §23.2) |
| Worker → Bedrock | Pregunta del contacto + knowledge del negocio; sin teléfonos | Rol IAM; perfil de inferencia EU; minimización (§20) |
| Worker → otros conectores (Google, SES, Meta, Telegram) | Solo los datos necesarios | Credenciales descifradas solo en memoria; timeouts y reintentos (§22.4) |
| Consola → datos | Solo con rol `operator` (`app_operator`) | Login con 2FA, IP permitida, cada acción auditada; las acciones que salen fuera van por la cola (`console_action`) (§4.1, §5.3, §23.2) |

**Qué nunca puede cruzar**

- Credenciales en claro: ni en Git, ni en logs, ni en imágenes, ni en respuestas HTTP (§5.3).
- Datos de un tenant en otro: RLS y claves foráneas compuestas `(tenant_id, x_id)` (§23.2).
- Teléfonos completos y tokens en logs, en argumentos de trabajos o en candados (§18.1, §5.3).
- Datos de salud en texto libre, formularios, prompts o packs (§20.2, §5.4).

**Cifrado**

- **En tránsito:** HTTPS en el ALB (80 redirige a 443) (§5.3).
- **En reposo:** RDS cifrado y buckets S3 cifrados (§11.4). Las credenciales de clientes se cifran con AES-256-GCM usando una clave de datos que KMS envuelve; rotación anual de la clave maestra activada en CDK (§5.1).

**Modelo de confianza entre módulos**

- Los packs acceden a los datos solo a través de la API de packs (`PackContext`); no tocan la base de datos ni las APIs directamente (§17, §19.3). Única excepción controlada: `packs/*/storage.py` (§4.2).
- `core/` no importa Django, `flowpilot/` ni `packs/`; un pack no importa otro pack (§4.2).
- Los conectores se llaman solo a través del proxy del registro, que resuelve binding y credencial (§22.1).
- La consola y el Django Admin solo los usa el rol operador (§8, §23.2).
- Las operaciones que escriben en sistemas externos son idempotentes (§18.4).

---

### 7. Configuración: qué vive dónde

> **En pocas palabras.** Hay tres sitios para guardar "ajustes" y cada cosa tiene uno solo. Así nunca hay dudas de dónde cambiar algo ni de si un cambio necesita desplegar.

| Dónde | Qué | Cómo se cambia | ¿Despliegue? |
|---|---|---|---|
| **Repositorio** | Código, packs, bloques, recorridos, parsers y validadores de datasets, plantillas de WhatsApp por defecto, manifests de conectores, migraciones, infraestructura (CDK) | PR + CI + revisión | Sí |
| **Base de datos** | Tenants, canales, bindings, credenciales, config de negocio, knowledge, datasets, asignación de pack, plantillas del tenant, formularios, watches de Calendar | Consola o Django Admin (con validación y auditoría) | **No** |
| **Entorno** (SSM → variables de entorno) | Lo que cambia entre local/staging/prod: URLs, secretos de plataforma, `META_GRAPH_VERSION`, modelo de IA por defecto, regiones de Bedrock y SES, niveles de log | CDK / consola de AWS | Reinicio de tareas (lo hace el despliegue) |

**Variables de entorno principales:**

```text
ENVIRONMENT=local|staging|prod
DJANGO_SECRET_KEY, DJANGO_ALLOWED_HOSTS, DJANGO_DEBUG=false
DATABASE_URL_RUNTIME
DATABASE_URL_OPERATOR                       (solo en la definición de tarea de web; el worker no la tiene)
DATABASE_URL_OWNER                          (solo en la tarea migrate)
PUBLIC_API_BASE_URL=https://api.<dominio>
GCAL_WEBHOOK_BASE_URL                       (por defecto = PUBLIC_API_BASE_URL; en local, la URL del túnel)
CREDENTIAL_CIPHER=kms|local, KMS_KEY_ID, LOCAL_CIPHER_KEY (solo local)
S3_BUCKET_EXPORTS, AWS_REGION                 ([FUTURO] S3_BUCKET_MEDIA)
META_GRAPH_VERSION=v2x.0
OWNER_BOT_TOKEN, OWNER_BOT_WEBHOOK_SECRET, OPS_TELEGRAM_CHAT_ID
BEDROCK_REGION, LLM_DEFAULT_MODEL           (id del perfil de inferencia EU; nunca en el código)
SES_REGION, SES_FROM
INTERNAL_METRICS_SECRET
SENTRY_DSN, LOG_LEVEL
WEB_WORKERS=2  WEB_THREADS=4
WORKER_QUEUES=inbound,outbound,scheduled,heavy   WORKER_CONCURRENCY=8
```

---

### 8. Consola y Django Admin

> **En pocas palabras.** Al principio, el **Django Admin** sirve para ver y editar datos, y una **consola mínima** (`consola.<dominio>`) añade solo lo que el Admin no hace bien: ver si todo va bien, reintentar trabajos, dar de alta un cliente, probar conversaciones y subir datasets. El resto de pantallas llegará cuando haga falta.

#### 8.1 Tecnología

Vistas de Django con plantillas + **HTMX** (actualizaciones parciales sin escribir JavaScript) + una hoja de estilos sencilla (Pico.css o similar). Sin frontend separado ni SPA. Autenticación de Django + 2FA. Permisos: `operator` (todo) y, en el futuro, `tenant_owner` (solo su tenant, vía RLS con `app_runtime`).

#### 8.2 Pantallas

| Pantalla | Contenido |
|---|---|
| **Django Admin** | Tenants, canales, bindings, config (validada contra el esquema del pack al guardar, con historial), knowledge, formularios, plantillas, tareas programadas, auditoría |
| **Salud** (inicio) | Estado general (verde/ámbar/rojo); trabajos pendientes y antigüedad del más antiguo por cola; errores en la última hora; conectores caídos; envíos `unknown`/`failed`; plantillas rechazadas; watches caducados; `side_effect` en `unknown` con "Marcar hecho" / "Reintentar" (§18.4); resultado de "Comprobar salud" por tenant |
| **Trabajos fallidos** | Trabajos `failed` por cola, con error; **Reintentar** o cancelar |
| **Alta simple de tenant** | Datos básicos → pack → config (formulario generado del esquema) → canales y conectores → comprobar salud → activar (§13.1) |
| **Simulador** | Canal de pruebas (§21.4): conversar con el bot de cualquier tenant, viendo estado, bloque actual y llamadas a conectores |
| **Subir dataset** | Elegir tenant y dataset → subir fichero en formato canónico → **informe del validador** (errores con fichero/fila/motivo) → guardar versión → activar (§19.5) |

Las acciones de consola que llaman a sistemas externos (comprobar salud, crear o renovar un watch, `setWebhook`, enviar plantillas a aprobación) **no se hacen en la petición**: encolan `console_action` (cola `scheduled`) y la página muestra su estado con HTMX (§4.1). El simulador no lo necesita: usa el camino normal de los eventos (§21.4).

[FUTURO]: buscador de conversaciones con `trace_id`, uso y costes por tenant, editor de knowledge con diff, asistente de alta completo, vista de revisión de datasets para el negocio, portal de cliente.

#### 8.3 Comprobación de salud de un tenant

Botón en la consola (vía `console_action`) y tarea diaria automática (`health_check_daily`, cola `heavy`). Comprueba:

- **canales**: token válido (llamada de lectura a la API), webhook configurado, plantillas aprobadas;
- **conectores**: el `health_check` de cada binding (p. ej. leer el calendario);
- **config**: valida contra el esquema actual del pack;
- **watch de Calendar** (si el pack usa `booking`): activo; ámbar si caduca en menos de 24 h (mismo umbral que la alarma, §12.4); última sincronización reciente;
- **datasets** (si el pack los declara): hay versión activa y el worker la carga.

El resultado se guarda en `tenant_health` y aparece en verde, ámbar o rojo por tenant.

---

### 9. Entornos y desarrollo local

> **En pocas palabras.** Hay tres entornos: tu portátil (local), uno de pruebas en AWS (staging) y el real (prod). Los tres ejecutan **la misma imagen**; solo cambia la configuración. **Nunca hay datos reales fuera de prod.**

| | Local | Staging | Prod |
|---|---|---|---|
| Dónde | Tu portátil, Docker Compose | Cuenta AWS de staging | Cuenta AWS de prod |
| BD | PostgreSQL 18 en un contenedor | RDS db.t4g.micro, Single-AZ | RDS db.t4g.small (→ Multi-AZ) |
| Cifrado | `LocalCipher` | KMS | KMS |
| Canales | Canal de pruebas (simulador), bot de Telegram de pruebas, número de prueba de Meta (vía túnel) | Canal de pruebas, número de prueba de Meta, bot de Telegram de staging | Reales; canal de pruebas solo para el operador |
| IA y email | `mock` (o Bedrock con tu rol); email a mailpit | Bedrock; SES en sandbox | Bedrock; SES |
| Datos | Semilla (`seed_demo`) | Ficticios | Reales |
| Despliegue | `docker compose up` | Automático en cada merge a `main` | Manual con aprobación |
| Coste | 0 | ~15–30 $/mes (se apaga fuera de horario) | §11.6 |

#### 9.1 Desarrollo local

```bash
uv sync                          # dependencias
make up                          # docker compose up -d db mailpit
make migrate seed                # esquema + tenants de demo (salon, bus, gym) con conectores mock y un dataset de ejemplo
make dev                         # web (runserver) + worker (procrastinate) con recarga automática
make tunnel                      # cloudflared/ngrok → URL pública para webhooks de Telegram/Meta y avisos de Calendar
make test                        # todos los tests;  make test-fast  (sin Postgres: core + packs)
make scenarios PACK=salon        # escenarios de un pack
make validate-dataset TENANT=demo-bus FILE=timetable.yaml   # validador de datasets en local
```

- `compose.yaml` local: `db` (PostgreSQL 18), `web`, `worker` y `mailpit` (para ver los emails que envía, p. ej., el pack `gym`; el adaptador `email` local es `smtp` hacia mailpit).
- Hay datos de demo para probar sin credenciales: los conectores `mock` devuelven huecos de calendario y respuestas de IA predefinidas, y el simulador (§21.4) permite conversar con los tres packs.

---

### 10. Construcción y despliegue

> **En pocas palabras.** Cada vez que se aprueba un cambio, se construye **una imagen** (un paquete con todo el programa) y se despliega primero en staging automáticamente. Para producción pulsas un botón. AWS arranca las copias nuevas, comprueba que funcionan y solo entonces apaga las viejas: **no hay cortes**. Si las nuevas fallan, vuelve solo a la versión anterior.

#### 10.1 La imagen

- Un único `Dockerfile` multi-etapa (dependencias con `uv` → imagen final mínima), **ARM64** (Graviton), Python 3.14.
- Etiqueta = SHA del commit. Se publica en **ECR** (cuenta de prod, compartida con staging mediante permisos entre cuentas).
- La misma imagen para `web`, `worker`, `migrate` y `smoke` (cambia solo el comando).

**Tenant `smoke`:** lo crea `ensure_smoke_tenant` (idempotente) en la tarea `migrate`, en staging y prod. Está **excluido** del reparto de periódicas, de `usage_daily`, de las alarmas por tenant y de `NotifyOwner`. Cada ejecución usa un contacto nuevo `smoke-<sha>` para no arrastrar estado.
- Los estáticos (CSS de la consola y de los formularios) se sirven con WhiteNoise desde la propia imagen.

#### 10.2 Pipeline (GitHub Actions)

```
on: pull_request  → job "ci": puertas §25.3

on: push a main   →
  1. build: construir la imagen ARM64 → push a ECR (tag = sha)
  2. deploy-staging (automático):
       a. asumir el rol de despliegue de staging (OIDC, sin claves guardadas)
          y arrancar RDS si está parado; esperar a que esté `available`
       b. ejecutar la tarea ECS "migrate" con la imagen nueva (migrate + sync_packs + ensure_smoke_tenant)
          y esperar a que termine bien
       c. actualizar los servicios web y worker con la nueva definición de tarea
       d. esperar a que el despliegue se estabilice (o se revierta solo)
       e. smoke test: GET /health + tarea ECS "smoke" (python manage.py smoke_test):
          una conversación por el canal de pruebas del tenant interno `smoke`
          (pack salon, conectores mock, contacto `smoke-<sha>`); espera la respuesta (máx. 60 s) y sale con 0/1
          en staging, si falla → el job falla y deploy-prod no se ofrece
          en prod, si falla → alarma 🔴 y rollback manual (§10.4)
  3. deploy-prod (manual: "environment: production" con aprobación requerida):
       mismos pasos a–e contra la cuenta de prod
```

#### 10.3 Cómo se despliega sin cortes

- **web**: despliegue progresivo de ECS con `minimumHealthyPercent=100`, `maximumPercent=200`. Arranca las tareas nuevas → el ALB comprueba `/health` → les envía tráfico → retira las viejas tras 30 s de *draining* (dejan de recibir peticiones nuevas y terminan las que tienen).
- **worker**: al pararse recibe SIGTERM → Procrastinate deja de coger trabajos nuevos y termina los que tiene → `stopTimeout=120 s` (**el máximo que permite Fargate**). Los pendientes siguen en la BD y los recoge la tarea nueva; los que se cortan se reintentan, por eso los trabajos largos son reanudables (§18.2).
- **Deployment circuit breaker con rollback automático**: si las tareas nuevas no pasan los health checks, ECS vuelve solo a la versión anterior.
- **Migraciones**: se ejecutan **antes** de actualizar los servicios. Como son compatibles con la versión anterior (§23.4), el código viejo sigue funcionando durante el despliegue.

#### 10.4 Vuelta atrás (rollback)

- **Automática**: el circuit breaker de ECS.
- **Manual**: re-ejecutar `deploy-prod` con el SHA anterior (botón en GitHub Actions). Las migraciones no se revierten (expand/contract las hace innecesarias).
- **Datos**: restauración a un momento concreto (PITR, §13.3), solo en caso de corrupción.

#### 10.5 Cuándo y cómo se despliega

- Staging: en cada merge.
- Prod: cuando quieras, **preferiblemente fuera de las horas punta de los clientes** (antes de las 9:00 o después de las 21:00). Con los despliegues sin corte no es obligatorio, pero reduce el riesgo.
- **Cambios de config y datasets de clientes: nunca necesitan despliegue.**
- Cambios de infraestructura: `cdk diff` en el PR (como comentario) → `cdk deploy` manual tras la aprobación, primero en staging.

---

### 11. Infraestructura en AWS

> **En pocas palabras.** Usamos servicios gestionados de Amazon: **ECS Fargate** ejecuta nuestro programa sin que tengamos que gestionar servidores (sube o baja el número de copias según la carga), **RDS** es una base de datos PostgreSQL que AWS mantiene (copias de seguridad, parches, réplica en otra zona), el **ALB** es la puerta de entrada con HTTPS, **Bedrock** da acceso a modelos de IA dentro de la UE y **SES** envía emails. Todo se define en código (CDK) para poder recrearlo idéntico. Empieza costando ~100 €/mes en producción y crece solo si hace falta.

#### 11.1 Cuentas y acceso

```
AWS Organizations (cuenta de gestión: solo facturación y organización, sin recursos)
 ├─ cuenta "staging"   → todo el entorno de pruebas
 └─ cuenta "prod"      → producción + ECR (registro de imágenes compartido)
Acceso humano: IAM Identity Center (SSO) + MFA. Sin usuarios IAM con claves.
CI: rol por cuenta asumible solo desde GitHub Actions de este repo (OIDC).
Presupuestos (AWS Budgets) con alertas por email al 50/80/100 % en cada cuenta.
```

#### 11.2 Región

**eu-south-2 (España, Aragón)** [VERIFICAR antes de crear nada]: que estén disponibles ECS Fargate (ARM), RDS PostgreSQL 18, ECR, KMS, SSM, CloudWatch, EventBridge (reglas programadas y *API destinations*), ALB y ACM. Precios similares a Irlanda; argumento comercial "datos en España". **Plan B: eu-west-1 (Irlanda)**, la región más completa.

- **SES**: el envío está disponible en eu-south-2 [VERIFICAR dominio verificado y salida del sandbox]. La recepción (SES Receiving) **no** lo está: solo afecta al email como canal [FUTURO] (§21.7).
- **Bedrock**: se usa con un perfil de inferencia geográfico EU (`BEDROCK_REGION`) [VERIFICAR que el perfil EU se puede invocar desde eu-south-2 con el modelo elegido, y a qué regiones destino enruta (hacen falta en la política IAM, §11.4); si no, `BEDROCK_REGION=eu-west-1` y el tráfico sigue en la UE].

#### 11.3 Red

```
VPC (2 zonas de disponibilidad)
 ├─ subredes públicas (2): ALB + tareas ECS (con IP pública, security group cerrado)
 └─ subredes privadas (2): RDS (sin acceso a internet)

Security groups:
  sg-alb    : entrada 443/80 desde internet
  sg-tasks  : entrada 8000 SOLO desde sg-alb; salida a internet (Meta, Telegram, Google, Bedrock, SES…)
  sg-db     : entrada 5432 SOLO desde sg-tasks
```

**[DECIDIDO] Sin NAT Gateway al principio** (cuesta ~33 $/mes + tráfico): las tareas salen a internet con su IP pública (3,65 $/mes cada una) y no aceptan conexiones salvo desde el ALB. Se pasa a subredes privadas + NAT cuando haya más de ~8 tareas o lo exija un cliente (§14).

#### 11.4 Servicios y tamaños iniciales

| Recurso | Configuración inicial (prod) | Staging |
|---|---|---|
| **ECS cluster** | Fargate, ARM64 | Igual |
| Servicio `web` | 0,5 vCPU / 1 GB, 1 tarea (mín. 1, máx. 4), gunicorn `gthread` 2 × 4; autoescala por CPU 60 % y por peticiones por tarea | 0,25 vCPU / 0,5 GB, 1 tarea |
| Servicio `worker` | 0,5 vCPU / 1 GB, 1 tarea (mín. 1, máx. 6), concurrencia 8; autoescala por `queue_oldest_age_s` (> 30 s → +1); sin datos = mantener el tamaño (`notBreaching`) | 0,25 vCPU / 0,5 GB |
| Tareas `migrate` y `smoke` | 0,5 vCPU / 1 GB, bajo demanda | Igual |
| **ALB** | 1, listener 443 (certificado ACM gratuito), reglas por host: `api.`, `consola.` (con filtro de IP origen) | Igual |
| **RDS PostgreSQL 18** | db.t4g.small (2 GB), 20 GB gp3 con crecimiento automático, cifrado, Single-AZ → **Multi-AZ** con 3–5 clientes de pago; backups 14 días + PITR; protección contra borrado | db.t4g.micro, 7 días de backup, se puede detener |
| **S3** | Bucket `exports` (cifrado, privado, ciclo de vida 7 días). [FUTURO] bucket `media` para ficheros recibidos | Igual |
| **KMS** | 1 clave para credenciales (**rotación anual activada en CDK**) + la de RDS/S3 gestionada por AWS | Igual |
| **ECR** | 1 repositorio, regla de ciclo de vida (guardar las últimas 30 imágenes) | (usa el de prod) |
| **CloudWatch** | Logs con retención de 30 días; métricas propias (EMF); alarmas → SNS | Retención de 7 días; **las alarmas no notifican** (sin acciones o `notBreaching`), porque staging se apaga |
| **EventBridge** | Regla programada cada minuto → *API destination* `POST https://api.<dominio>/internal/metrics/queues`; *Connection* de tipo `API_KEY` (cabecera con el secreto, guardado por AWS en Secrets Manager); `MaximumRetryAttempts=0`, `MaximumEventAgeInSeconds=60` (§12.3). Otra regla programada vuelve a parar el RDS de staging cada noche (RDS se arranca solo a los 7 días parado) | Igual |
| **Bedrock** | Perfil de inferencia EU. Permiso `bedrock:InvokeModel` en el rol de la tarea `worker` sobre **el ARN del perfil y** sobre `arn:aws:bedrock:<cada región destino del perfil>::foundation-model/<modelo>` (con perfiles entre regiones hacen falta los dos) | Igual |
| **SES** | Solo envío; dominio verificado (SPF, DKIM, DMARC); remitente `SES_FROM` | Sandbox |
| **Route 53** | Zona del dominio; `api.`, `consola.` | `*.staging.<dominio>` |

**Detalles importantes:**

- **Conexiones a la BD:** sin RDS Proxy; cálculo por tarea en §4.1.
- **Health check:** `/health` comprueba que el proceso responde y que hay conexión a la BD (sin llamar a servicios externos).
- **Fargate sin acceso SSH:** para depurar se usa **ECS Exec** (consola dentro del contenedor, auditada), solo con tu rol.

#### 11.5 Infraestructura como código (CDK en Python)

```
infra/
  app.py                       # define staging y prod con sus parámetros
  stacks/
    network_stack.py           # VPC, subredes, security groups
    data_stack.py              # RDS, KMS (con rotación), S3, parámetros SSM
    registry_stack.py          # ECR (solo prod) + permisos para staging
    app_stack.py               # cluster ECS, definiciones de tarea, servicios, ALB, autoescalado, DNS, SES, permisos de Bedrock
    observability_stack.py     # regla de EventBridge + API destination, alarmas, SNS, Lambda de alertas a Telegram, dashboard
    ci_stack.py                # proveedor OIDC de GitHub + roles de despliegue
  config/
    staging.py  prod.py        # tamaños, número de tareas, Multi-AZ sí/no, dominios
```

- Cambiar de Single-AZ a Multi-AZ, o de 1 a 2 tareas web = cambiar un valor en `prod.py` → PR → `cdk deploy`.
- Recursos con datos (RDS, buckets, clave KMS) con **protección contra borrado** y `RemovalPolicy.RETAIN`.

#### 11.6 Costes [ESTIMACIÓN con precios de referencia de 2026; verificar en la calculadora de AWS para la región elegida]

| Pieza | Prod inicial | Prod con alta disponibilidad (3–5 clientes de pago) |
|---|---:|---:|
| Fargate ARM (web + worker) | ~32 $ | ~48 $ (2 web) |
| IPv4 públicas (tareas + ALB) | ~15 $ | ~18 $ |
| ALB | ~20 $ | ~20 $ |
| RDS db.t4g.small + almacenamiento + backups | ~30 $ | ~58 $ (Multi-AZ) |
| KMS, SSM, CloudWatch, ECR, S3, Route 53, SES | ~8 $ | ~10 $ |
| Bedrock (por uso: solo FAQ de `salon`) | ~1–5 $ | ~5–10 $ |
| EventBridge (1 llamada por minuto) + secreto de la *Connection* | ~0,40 $ | ~0,40 $ |
| **Total prod** | **~110 $/mes (~100 €)** | **~165 $/mes (~150 €)** |
| Staging (apagado fuera de horario) | ~15–30 $/mes | ~15–30 $/mes |

**Para reducirlo:** créditos de cuenta nueva (100–200 $); **AWS Activate Founders** (1.000 $, tras el alta); **no crear prod hasta el primer cliente real**; Savings Plans (−20–40 %) cuando el uso sea estable; staging apagado de noche y en fines de semana (tareas a 0, RDS detenido).

#### 11.7 Fases de infraestructura

| Fase | Qué hay |
|---|---|
| **Desarrollo** | Local + staging bajo demanda (créditos) |
| **Primer cliente real** | Prod inicial (Single-AZ, 1 web + 1 worker) |
| **3–5 clientes de pago** | RDS Multi-AZ + 2 tareas web en 2 zonas |
| **20+ clientes o caídas caras** | Subredes privadas + NAT, WAF, worker `heavy` separado, copia de backups a otra región, Savings Plans |

---

### 12. Observabilidad y alertas

> **En pocas palabras.** Saber en todo momento si algo va mal **antes de que te llame un cliente**, y poder reconstruir cualquier conversación en minutos.

#### 12.1 Logs

- **JSON estructurado** a la salida estándar → CloudWatch Logs.
- Campos obligatorios: `ts`, `level`, `msg`, `env`, `service` (web/worker), `tenant_id`, `trace_id`, `event_id`, `conversation_id`, `job`.
- `trace_id` = id del `inbound_event` que originó todo; se propaga a ejecuciones, envíos, llamadas a conectores y logs. Buscar por `trace_id` muestra la historia completa.
- Sin datos sensibles (§5.3).

#### 12.2 Errores

**Sentry** (plan gratuito al principio): excepciones de web y worker con el contexto (`tenant_id`, `trace_id`), y versiones por SHA de despliegue.

#### 12.3 Métricas

**Métrica de la cola [DECIDIDO]:** la publica `web`, no el worker (un worker caído no publicaría su propia alarma). Una regla de EventBridge llama cada minuto a `POST /internal/metrics/queues` (cabecera con `INTERNAL_METRICS_SECRET`); la vista consulta la tabla de trabajos de Procrastinate (pendientes ya vencidos por cola y antigüedad del más antiguo, fallidos recientes) y escribe el resultado como **log EMF**, que CloudWatch convierte en métrica.

- Solo cuenta **trabajos que se pueden coger ya**: `status='todo'`, `scheduled_at <= now()` (o sin programar) y sin otro trabajo anterior con el mismo `lock` en `todo` o `doing` (los que esperan a su candado no indican falta de workers).
- La **alarma** trata la ausencia de datos como fallo (`TreatMissingData=breaching`): si la métrica deja de llegar, algo está roto. El **autoescalado** la trata como `notBreaching` (mantiene el tamaño, no escala a ciegas).
- La regla de EventBridge no reintenta (`MaximumRetryAttempts=0`, `MaximumEventAgeInSeconds=60`): un minuto perdido no se recupera, se ve como hueco.

| Métrica | Origen | Uso |
|---|---|---|
| `queue_pending{queue}`, `queue_oldest_age_s{queue}` | `web` vía EventBridge (cada 60 s) → EMF | **Autoescalado del worker** y alarmas |
| `jobs_failed_total{queue}` | Ídem | Alarma |
| `webhook_latency_ms` (p50/p95) | Middleware → EMF | Alarma |
| `send_status{status}` (sent/failed/unknown) | Ídem | Alarma |
| `connector_calls{category,adapter,result}`, `breaker_open` | Registro de conectores | Consola + alarma |
| `gcal_sync{result}`, `gcal_watch_expires_in_h` | Sincronización y renovación de watches | Alarma |
| `dataset_upload{result}`, `dataset_load{result}` | Consola (subida) y worker (carga) | Alarma |
| `llm_tokens{tenant}`, `cost_eur{tenant,kind}` | `usage_daily` | Consola y cuotas |
| CPU y memoria por servicio, conexiones de RDS, CPU de RDS, espacio libre | AWS (automático) | Alarmas y autoescalado |

#### 12.4 Alarmas → Telegram (ops) + email

| Alarma | Umbral inicial | Gravedad |
|---|---|---|
| `/health` del ALB sin tareas sanas | 1 min | 🔴 |
| `queue_oldest_age_s` inbound (**sin datos = alarma**) | > 60 s durante 3 min | 🔴 |
| `jobs_failed_total` | > 5 en 10 min | 🟠 |
| Envíos `failed` + `unknown` | > 3 en 15 min | 🟠 |
| Circuit breaker abierto (cualquier tenant) | 1 | 🟠 |
| Watch de Calendar caducado o sincronización fallando | Watch con < 24 h sin renovar, o 3 sincronizaciones fallidas seguidas, o > 2 h sin sincronizar | 🟠 |
| Dataset rechazado | Subida rechazada por el validador: 🟡 · el worker no puede cargar el dataset activo: 🔴 | 🟡 / 🔴 |
| Errores 5xx del ALB | > 1 % durante 5 min | 🟠 |
| RDS: CPU > 80 % / espacio libre < 20 % / conexiones > 80 % | 10 min | 🟠 |
| Plantilla de WhatsApp rechazada / calidad del número baja | 1 | 🟡 |
| Número de WhatsApp > 800 mensajes de servicio en el mes | 1 por número | 🟡 (aviso de coste al cliente) |
| Smoke test fallido en prod | 1 | 🔴 (rollback manual, §10.4) |
| Despliegue revertido automáticamente | 1 | 🟠 |
| Presupuesto de AWS al 80 % | 1 | 🟡 |
| `ops_alert{kind}` (token caducado, credencial revocada, cuota de IA superada, ventana cerrada sin plantilla…) | ≥ 1 en 5 min | 🟠 |

La alarma de 800 mensajes de servicio (**por número**, no por tenant) responde al cambio de precios de Meta: desde el 1/10/2026, los mensajes de servicio se cobran al precio *utility* del país por encima de 1.000 al mes por número. Fuentes: Meta, *Pricing for non-template messages*: https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing/non-template-messages (cobro por mensaje de servicio desde el 1/10/2026, a la tarifa del mercado). El tramo gratuito de **1.000 mensajes al mes por número** no aparece en esa página [VERIFICAR]; lo recogen https://techweez.com/2026/09/28/whatsapp-business-pricing-october-2026/ y https://www.courier.com/blog/whatsapp-pricing-changes-october-2026. Ver §21.2.

En **staging** las alarmas existen pero no notifican (staging se apaga cada noche).

**La aplicación nunca avisa a ops directamente**: emite la métrica `ops_alert{kind}` (EMF) y la alarma hace el resto.

Camino: alarma de CloudWatch → SNS → (email) + Lambda mínima que publica en el chat de ops de Telegram. **Las alarmas no dependen de que la plataforma esté viva** (si se cae la web, falta la métrica de la cola y la alarma salta igualmente).

#### 12.5 Panel

Dashboard de CloudWatch (definido en CDK) con las métricas de §12.3 + la pantalla de salud de la consola (§8.2). Uptime externo (Better Stack o UptimeRobot, gratis) contra `https://api.<dominio>/health` desde fuera de AWS.

---

### 13. Operación técnica

> **En pocas palabras.** Cómo dar de alta y de baja a un cliente, hacer y probar copias de seguridad, y qué hacer cuando algo falla.

#### 13.1 Alta de un tenant

1. **Consola → Alta simple de tenant** (§8.2): datos básicos, pack, config.
2. **WhatsApp** (modo `client_app`, §21.2): el operador hace el alta **dentro del Business Portfolio del cliente** siguiendo la checklist de §21.2 → introduce `app_id`, `app_secret`, token permanente, `waba_id` y `phone_number_id` → la consola muestra la URL del webhook de la app y el `verify_token` (en claro solo esta vez) para configurarlos → se suscribe el campo `messages` y la app a la WABA (`POST /<WABA_ID>/subscribed_apps`). **Plantillas: crearlas cuanto antes** (tardan de 1 a 48 h en aprobarse).
3. **Telegram** (si el tenant lo usa como canal): bot creado con @BotFather a nombre del cliente → token en la consola → `setWebhook` (vía `console_action`).
4. **Avisos al dueño**: `owner_channel` = Telegram → enlace de vinculación con el bot de plataforma (§21.6).
5. **Google Calendar** (packs con `booking`): el cliente **comparte su calendario con el email de la cuenta de servicio** con permiso "Hacer cambios en los eventos" → binding con el `calendar_id` → **crear el watch** (botón en la consola; §22.2).
6. **Datasets** (packs que los declaran, p. ej. `bus`): convertir los datos del cliente al formato canónico con la herramienta de `tools/` → **Consola → Subir dataset** → activar.
7. **Formulario** (packs con handler `form`, p. ej. `gym`): crear el `form` → enviar la URL pública al cliente.
8. **Comprobar salud** en verde (§8.3).
9. Prueba con el **simulador** y después prueba real con el dueño.
10. Activar. Se registra en `audit_event`.

#### 13.2 Baja de un tenant

Comando o botón `offboard_tenant <slug>`:

1. Estado `offboarded` → los webhooks se contestan con 200 sin guardar nada (§18.1); los formularios dejan de aceptar envíos.
2. Cancelar las tareas programadas, **parar el watch de Calendar** (`channels.stop`) y llamar a `deleteWebhook` de cada bot de Telegram del tenant (**antes** de borrar sus credenciales).
3. `export_tenant` (si se ha acordado con el cliente) → zip en S3 `exports` con un enlace temporal.
4. Borrado de datos personales y credenciales (se conservan las métricas agregadas y la auditoría).
5. Checklist manual: el cliente revoca el acceso en su Meta App y deja de compartir el calendario; verificar.
6. `audit_event`.

#### 13.3 Backups y restauración

| Qué | Cómo | Retención |
|---|---|---|
| RDS | Backups automáticos + **PITR** (restaurar a cualquier momento de los últimos 14 días) | 14 días |
| RDS (largo plazo) | Snapshot semanal copiado a otra región [FUTURO en la fase de 20+ clientes] | 3 meses |
| S3 (`exports`) | — | 7 días |
| Config y datasets del tenant | `config_history` y `dataset_version` en la BD | Indefinida |
| Infraestructura | CDK en Git | — |

**Simulacro de restauración trimestral** (manual, runbook 6): restaurar el último backup en una instancia temporal de la cuenta de prod → script de comprobación (conteos por tenant con `app_operator`, últimas filas) → borrar la instancia → anotar el resultado. Objetivos: **RPO ≤ 5 min** (PITR), **RTO ≤ 2 h**. [FUTURO] automatizarlo.

#### 13.4 Runbooks (en `docs/runbooks/`)

1. El bot no responde a nadie.
2. El bot no responde a un tenant concreto (token caducado, webhook desconfigurado, número con calidad baja).
3. La cola se acumula (o falta la métrica de la cola).
4. Un conector caído (Google, Bedrock, SES).
5. Envíos en estado `unknown`.
6. Restaurar la BD (PITR) y simulacro trimestral.
7. Rollback de un despliegue.
8. Incidente de seguridad o fuga de datos (pasos y plazos de aviso al cliente).
9. Rotar credenciales de un tenant.
10. Watch de Calendar caducado o sincronización rota (recrear el watch, forzar sincronización completa, comprobar que el calendario sigue compartido).
11. Dataset rechazado por el validador (leer el informe, corregir con la herramienta de conversión, volver a subir; si falla la carga del activo, volver a la versión anterior).

---

### 14. Escalado

> **En pocas palabras.** Al principio una copia de cada cosa basta de sobra. Si crece la carga, AWS añade copias solo. Esta tabla dice qué vigilar y qué cambiar en cada caso, **sin rediseñar nada**.

**Capacidad de partida [ESTIMACIÓN]:** 50 negocios ≈ 75.000 mensajes al mes, con picos de 1–2 por segundo. Un worker con concurrencia 8 procesa 20–50 trabajos por segundo → **margen de 10 a 50 veces**.

| Señal | Acción | Coste/esfuerzo |
|---|---|---|
| `queue_oldest_age_s` > 30 s sostenido | El autoescalado añade workers (máx. 6) | Automático |
| Trabajos `heavy` retrasan a `inbound` | Separar un servicio `worker-heavy` (misma imagen, `WORKER_QUEUES=heavy`) | Cambio en CDK |
| CPU de web > 60 % | El autoescalado añade tareas web | Automático |
| Conexiones de RDS > 80 % | Rehacer el cálculo de §4.1; bajar hilos o subir la clase de RDS | Configuración |
| CPU de RDS > 70 % sostenido | Subir la clase (t4g.small → t4g.medium → m7g) | Unos minutos de corte (con Multi-AZ, ~1 min) |
| Muchas lecturas de la consola o informes | Réplica de lectura de RDS para la consola | Cambio en CDK + router de Django |
| Más de ~8 tareas | Subredes privadas + NAT (en lugar de IPs públicas) | Cambio en CDK |
| Postgres como cola al límite (muy lejano: miles de trabajos/s) | Cambiar la cola a **SQS** detrás de la misma interfaz, con patrón *outbox* | Proyecto acotado |
| Un tenant muy grande | Límite de concurrencia por tenant; en el extremo, worker dedicado con cola propia | Configuración |
| Límites de Meta (por número o por usuario) | Control de ritmo en `outbound` (ya previsto) | — |

---

### 15. Estructura del repositorio

```
flowpilot/
├── pyproject.toml · uv.lock · manage.py · Dockerfile · compose.yaml · Makefile
├── CLAUDE.md                         reglas del framework dev-team + "Reglas de FlowPilot" para agentes
├── IDEA.md · design.md · spec.md · plan.md
├── devteam.config.yml
├── src/
│   ├── config/                       settings/{base,local,staging,prod}.py · urls.py · wsgi.py
│   ├── core/                         ── Python puro, sin Django ──
│   │   ├── domain/                   InternalMessage, eventos de dominio (MessageReceived, FormSubmitted…), Reply, Action
│   │   ├── packs/                    Pack, Periodic, Journeys, Step, PackContext, HandleResult (contratos, sin descubrimiento)
│   │   ├── conversation/             motor de recorridos, StepResult (Ask/Done/Back/Abort/Handoff/NotUnderstood), comandos globales
│   │   ├── blocks/                   bloques comunes  ·  blocks/booking/  bloques de reservas + slots.py (compute_slots)
│   │   ├── datasets/                 contrato Dataset (parser + validador), informe de errores
│   │   ├── connectors/interfaces/    booking.py, llm.py, email.py
│   │   ├── channels/                 capabilities, degradación, plan_sends (fusión, ventana 24 h)
│   │   └── testing/                  runner de escenarios, mocks programables
│   ├── flowpilot/                    ── Django ── (§4.2)
│   │   ├── tenancy/  channels/{whatsapp,telegram,test,owner_bot}/  forms/  calendars/  datasets/  packs_registry/
│   │   ├── connectors/{registry,adapters/{google_calendar,bedrock,ses,smtp,mock}}/
│   │   ├── conversations/  events/  scheduling/  credentials/  usage/  audit/  ops/  console/
│   │   └── jobs.py                   app de Procrastinate + definición de tareas
│   └── packs/
│       ├── salon/   {pack.py, config.py, journeys.py, blocks/, handlers.py, templates/, knowledge/, scenarios/}
│       ├── bus/     {pack.py, config.py, journeys.py, blocks/, datasets/ (parser + validador), engine/, matcher/, scenarios/}
│       └── gym/     {pack.py, config.py, handlers.py, scenarios/}
├── tools/                            conversores de un solo uso por cliente (p. ej. Excel → timetable canónico)
├── tests/                            integration/ · isolation/ · contract/ (los unit y escenarios viven junto al código)
├── tests/fixtures/                   respuestas grabadas de APIs, datasets de ejemplo
├── infra/                            CDK (§11.5)
├── tasks/ · context/                 flujo de trabajo de dev-team
├── docs/
│   ├── adr/                          ADR-001 … ADR-005
│   ├── packs/                        un documento por pack
│   ├── runbooks/
│   └── api.md
└── .github/workflows/                ci.yml · deploy.yml
```

[FUTURO] `tenants/<slug>/` (recetas propias, variantes y escenarios por tenant) cuando exista la primera receta por tenant (§19.6).

---

### 16. Pendiente de verificar y decisiones abiertas

**Abierto:**

| # | Tema | Qué falta | Cuándo |
|---|---|---|---|
| 1 | Región eu-south-2 | [VERIFICAR] servicios (ECS ARM, RDS PostgreSQL 18, EventBridge *API destinations*, SES envío, perfil EU de Bedrock invocable desde la región) y precios | Antes de crear la infraestructura |
| 2 | Límites de envío de Meta | [VERIFICAR] rendimiento por número y límite por par usuario-negocio, para el control de ritmo de `outbound` | Canal WhatsApp |
| 3 | Watch de Google Calendar | [VERIFICAR] que `events.watch` funciona con cuenta de servicio sobre un calendario compartido, y su TTL real (máx. ~30 días) | Pack `salon` |
| 4 | SES | [VERIFICAR] dominio verificado y salida del sandbox en la región | Pack `gym` |
| 5 | Aviso de IA (AI Act art. 50) | [VERIFICAR] alcance exacto (¿también bots sin IA?) | Pack `salon` |
| 6 | Retención definitiva | Ajustar §23.3 al DPA firmado | Antes del primer cliente de pago |
| 7 | Tramo gratuito de Meta | [VERIFICAR] los 1.000 mensajes de servicio gratis al mes por número en la documentación oficial (§12.4) | Canal WhatsApp |
| 8 | Bedrock | [VERIFICAR] regiones destino del perfil EU para la política IAM (§11.4) | Pack `salon` |

**Cerrado:**

| Tema | Decisión |
|---|---|
| IA por defecto | Amazon Bedrock con perfil de inferencia EU; modelo por configuración (§20) |
| Versiones | **Python 3.14** (Django 5.2 lo soporta) · **Django 5.2 LTS** (soporte hasta abril de 2028; migrar a 6.2 LTS cuando salga, en abril de 2027) · **PostgreSQL 18** (disponible en RDS; PK con UUIDv7 nativo) · **Procrastinate 3.x** |

**[FUTURO]** (se decidirá cuando haga falta):

| Tema | Nota |
|---|---|
| Tech Provider / Embedded Signup | Implementar `platform_app` en WhatsApp (§21.2) tras el alta y la verificación de la empresa |
| Señal o prepago | Bloque `request_deposit` con una pasarela (Stripe Payment Links o Bizum vía TPV) cuando un cliente lo pida |
| Portal de cliente | Alcance y permisos, cuando el bot del dueño no baste |
| Email como canal | Elegir región de SES Receiving (eu-west-1) o un proveedor de *inbound parse* (§21.7) |
| Varios profesionales por negocio | `choose_staff` y `staff_id` en el conector `booking` (un calendario por profesional) |
| Ficheros recibidos | Descargar audios, imágenes y documentos a un bucket `media` (hoy se responde "escríbelo") |
| Acceso de dueños a la web | `tenant_membership` (usuarios Django por tenant) y portal |

---

## Module Contracts

### 17. Runtime de eventos (el corazón)

> **En pocas palabras.** Todo lo que llega (un mensaje, un formulario, un cambio en el calendario, una tarea que vence, un tic periódico) se convierte en un **evento guardado** y pasa por el mismo camino: cargar datos → ejecutar el pack → guardar el resultado. Nunca se deja una transacción abierta mientras se habla con un sistema externo.

**Eventos de dominio** (en `core/domain/`; es lo que recibe un handler):

| Evento | Origen | Handler |
|---|---|---|
| `MessageReceived(message: InternalMessage)` | WhatsApp, Telegram, canal de pruebas | `message` (motor de conversación) |
| `PayloadReceived(prefix, value, message)` | Botón con payload `<prefijo>:<valor>` (p. ej. `reminder_cancel:<gcal_event_id>`) | `payload:<prefijo>` (§19.3) |
| `FormSubmitted(form_id, fields, consent_at)` | Formulario público | `form` |
| `CalendarChanged(binding_id, gcal_event_id, event: CalendarEvent, change: created/updated/cancelled)` | Sincronización de Google Calendar (§22.2) | `calendar` |
| `TaskDue(key, task_name, payload)` | `scheduled_task` que vence | `task:<nombre>` |
| `PeriodicTick(name, scheduled_at)` | Tarea periódica del pack | la de `periodic` |

Firma de todo handler: `handler(event, ctx: PackContext) -> HandleResult`.

**Todo pasa por `inbound_event`** (también lo interno): cada ejecución periódica crea `InboundEvent(source=system, provider_event_id=periodic:<pack>:<nombre>:<instante programado>)` por tenant, y cada vencimiento de tarea, `InboundEvent(source=system, provider_event_id=task:<scheduled_task_id>)`. Así una periódica que se reintenta o se lanza dos veces no se ejecuta dos veces, y todo queda trazado con su `trace_id`.

**[DECIDIDO] Transacciones cortas.** Ninguna transacción queda abierta mientras se llama a un sistema externo (Calendar, Bedrock, Meta): una llamada lenta no debe retener conexiones ni bloqueos.

```python
# flowpilot/tenancy/context.py
@contextmanager
def tenant_tx(tenant_id):
    """Transacción corta con RLS: abre la transacción y fija app.tenant_id solo para ella."""
    assert not connection.in_atomic_block, "tenant_tx nunca se anida"   # (§23.2)
    with transaction.atomic():
        with connection.cursor() as c:
            c.execute("SELECT set_config('app.tenant_id', %s, true)", [str(tenant_id)])  # = SET LOCAL
        yield
```

```python
# flowpilot/events/runtime.py  (pseudocódigo)
def process_event(tenant_id, event_id):                     # el trabajo recibe solo ids (§18.2)
    # tx1: cargar
    with tenant_tx(tenant_id):
        event = InboundEvent.get(event_id)
        if event.status != "pending":
            return                                          # ya procesado (reintento tras commit)
        tenant = load_tenant(tenant_id)
        assignment = tenant.pack_assignment                 # qué pack y qué config
        conversation = load_or_open_conversation(event)     # None si el evento no es de conversación
        snapshot = Snapshot(tenant, assignment, conversation, conversation and conversation.version,
                            active_knowledge(), active_dataset_versions())

    # sin transacción: aquí van las llamadas externas (Calendar, Bedrock…)
    pack = registry.get(assignment.pack_name)
    ctx = build_context(snapshot, connectors_proxy(tenant_id, channel), dataset_cache, clock)
    result = dispatch(pack, to_domain_event(event), ctx)    # → replies + actions + nuevo estado

    # tx2: persistir
    with tenant_tx(tenant_id):
        if conversation and not persist_state(conversation, result.state, expected_version=snapshot.version):
            raise ConversationChanged                       # UPDATE … WHERE version=:v no tocó filas → reintento
        intents = plan_sends(result.replies, channel)       # fusión de textos, ventana de 24 h, degradación
        apply_actions(result.actions)                       # tareas, handoff, NotifyOwner, SendTemplate, UpdateContact
        record_execution(event, result, timings, costs)
        event.mark_done()
        defer_sends(intents)                                # el trabajo solo existe si hay COMMIT
```

- Si el worker muere entre tx1 y tx2, el trabajo se reintenta desde el principio: las lecturas se repiten y las escrituras externas ya hechas no se duplican gracias a `side_effect` (§18.4).
- **Quién escribe una conversación.** Todo trabajo que lea o escriba una conversación toma su candado `conv:<h>` (§18.1): `process_event`, `run_task` de tareas con `conversation_id`, y `process_owner_update` cuando el dueño responde a una derivación o pulsa "Devolver al bot". `human_until` no tiene trabajo propio: `process_event` lo comprueba de forma perezosa (si ha vencido, la conversación vuelve a modo bot antes de procesar el evento). Además, `conversation.version` se incrementa en cada escritura y tx2 hace `UPDATE … WHERE version = :v`: si otro escritor se coló (p. ej. la consola), tx2 no escribe nada y el trabajo se reintenta.
- `dispatch(pack, domain_event, ctx)` (en `flowpilot/events`) elige el handler por la clave de §19.1 (`message`, `payload:<prefijo>`, `form`, `calendar`, `task:<nombre>`, periódica). `"conversation"` significa el motor de `core/conversation` con `pack.journeys`.
- Los avisos de estado de WhatsApp no llegan a `dispatch` (§18.2). `process_event` comprueba antes `human_until` (§18.7).
- Los eventos sin conversación (formulario, cambio de calendario, tarea, periódica) siguen el mismo esquema con `conversation = None`.
- `SendTemplate` (envío a alguien sin conversación abierta, p. ej. confirmación de una cita manual) lo resuelve la plataforma en `apply_actions` (§21.1).

Todo pack recibe **solo** un `PackContext` (§19.3) y devuelve un `HandleResult`. **Nunca toca la BD ni las APIs directamente.**

---

### 18. Fiabilidad: que no se pierda ni se duplique nada

> **En pocas palabras.** Internet falla, los servidores se reinician y Meta a veces envía el mismo aviso dos veces. El sistema está hecho para que en esos casos **ningún evento se pierda** (todo se guarda antes de procesarse) y **nada se haga dos veces** (cada operación importante tiene una "clave" que impide repetirla).

#### 18.1 Entrada idempotente

```
POST /webhooks/<canal>/<public_id>
  1. resolver la entrada (funciones SECURITY DEFINER, §23.2)
       Telegram: public_id → canal → tenant
       WhatsApp: public_id → app (para elegir el app_secret); el canal de cada elemento se resuelve
                 después por metadata.phone_number_id (§21.2)
       no existe                       → 404
  2. verificar firma                   → 401 sin guardar nada si falla
  3. canal o tenant offboarded         → 200 sin guardar nada
     canal o tenant paused             → guardar el evento con status=ignored y contestar 200
                                         (si no, Meta y Telegram reintentan durante días)
     tenant onboarding                 → solo se procesa el canal test; el resto, como pausado
  4. tenant_tx(tenant_id):
       INSERT inbound_event … ON CONFLICT (tenant_id, source, provider_event_id) DO NOTHING
       si se insertó: procrastinate.defer(process_event, queue="inbound",
                                          lock=f"conv:{h}" solo si es de conversación; si no, sin candado)
     COMMIT
  5. 200 OK
```

- `h` = **sha256 truncado** (16 hex) de `(channel_id, contact_external_id)`. **El candado nunca lleva el teléfono** en claro: la tabla de trabajos de Procrastinate no está cifrada a nivel de campo ni tiene RLS.
- `provider_event_id` según la fuente: tabla de §2.2. `channel_id` puede ser NULL (formularios, cambios de calendario, `system`), por eso la unicidad es por `(tenant_id, source, provider_event_id)`.
- Formularios (`/f/…`) y simulador siguen los mismos pasos con su propia verificación (§5.3). Los avisos de Google Calendar **no** crean `inbound_event`: solo encolan una sincronización (§22.2).

#### 18.2 Colas y trabajos

| Cola | Trabajos | Candado (lock) | Reintentos |
|---|---|---|---|
| `inbound` | `process_event(tenant_id, event_id)`, `process_owner_update(tenant_id, event_id)` | `conv:<h>` en eventos de conversación → orden por conversación; sin candado el resto (formularios, calendario, avisos de estado, órdenes del dueño que no van a una conversación) | 5, backoff exponencial (2 s → ~1 min) |
| `outbound` | `send_message(tenant_id, intent_id)`, `reconcile_send(tenant_id, intent_id)`, `notify_owner(tenant_id, notification_id)` | `out:<conversation>` → se envían en orden; `notify_owner` usa `owner:<tenant>` | Según el error (§18.3) |
| `scheduled` | `run_task(tenant_id, scheduled_task_id)`, reparto de periódicas, `sync_calendar(tenant_id, binding_id)`, `renew_watches`, `poll_calendars`, `console_action(action_id)` | `conv:<h>` si la tarea tiene `conversation_id`; si no, `sched:<key>`; `sync_calendar` usa `queueing_lock=gcal:<binding>` (agrupa avisos) | 3 |
| `heavy` | `retention_cleanup`, `export_tenant`, `health_check_daily` | `export_tenant`: `heavy:<tenant>` (una a la vez por tenant). `retention_cleanup` y `health_check_daily`: `queueing_lock` con el nombre del trabajo; procesan cada tenant con su `tenant_tx` | 2 |

- Los **avisos de estado** (WhatsApp) los procesa `process_event` sin pasar por el pack (actualiza `send_intent`, `message` y coste), sin candado.
- **Una sola copia de worker atiende las 4 colas al principio.** Se separan en servicios distintos cuando haga falta (§14), sin tocar código.
- Trabajo que agota sus reintentos → estado `failed` en Procrastinate → visible en **Consola → Trabajos fallidos**, con botón "Reintentar" → alarma (§12.4).
- Los trabajos reciben **solo ids**, nunca datos personales en los argumentos ni en el candado.
- **Trabajos que recorren tenants** (reparto de periódicas, `renew_watches`, `poll_calendars`, `retention_cleanup`, salud diaria): obtienen la lista con funciones `SECURITY DEFINER` que solo devuelven ids (`fp_active_tenants(pack)`, `fp_watches_expiring(hours)`, `fp_bindings_to_poll()`) y procesan cada tenant con su propia `tenant_tx` (§23.2).
- **Trabajos `heavy` reanudables [DECIDIDO]:** al parar una tarea, Fargate da como máximo **120 s** antes de matarla (§10.3). Todo trabajo que pueda durar más se hace **por lotes con el progreso guardado** (p. ej. `retention_cleanup` borra de 1.000 en 1.000 y anota el último id) o es idempotente de punta a punta, de modo que al reintentarse continúa donde estaba.

#### 18.3 Envío de mensajes

```
send_message(intent_id):
  intent.status == sent              → terminar (ya se envió)
  intent.status == unknown           → no reenviar automáticamente (ver abajo)
  intent.status == sending           → un intento anterior se cortó a mitad → status = unknown
  status = sending; attempts += 1; COMMIT
  enviar (WhatsApp: con biz_opaque_callback_data = intent_id)
  respuesta del proveedor:
    OK                               → status = sent, provider_message_id = wamid / message_id; crear message(out)
    4xx (no 429)                     → status = failed (sin reintento); si es el token → ops_alert (§12.4)
    429, o 5xx con cuerpo de error   → status = pending; reintento con backoff (el proveedor NO lo aceptó)
    502/504 sin cuerpo, timeout, red → status = unknown (no sabemos si se aceptó)
```

**Conciliación de `unknown`** (trabajo `reconcile_send`, que `send_message` encola a los 5 min al pasar a `unknown`):

- **WhatsApp**: busca un aviso de estado cuyo `biz_opaque_callback_data` sea el `intent_id` → `sent`. Si no hay → alerta en la consola; reintento manual.
- **Telegram** (no hay avisos de estado): se reenvía solo si `Reply.safe_to_repeat` (por defecto `True` en `Text`, `Buttons` y `List`; `False` en `Template`); si no, alerta y reintento manual.
- Un `unknown` en la posición N **no bloquea** el envío N+1 de la misma conversación (`out:<conversation>`): el orden se respeta entre los que sí se envían.

Garantía: **al menos una vez + idempotencia**. Nunca se promete "exactamente una vez". No se reintenta a ciegas un envío ambiguo.

#### 18.4 Efectos externos idempotentes

- **Claves** (`ctx.idem`):
  - `ctx.idem.key("booking:create")` → `"<tenant>:<event_id>:booking:create"` (`event_id` = `inbound_event.id`): ligada al evento actual; estable si el mismo evento se reprocesa. Formularios: `ctx.idem.key("form_email")`; avisos al dueño: la plataforma usa `key("notify:<kind>")`.
  - `ctx.idem.stable("reminder:<gcal_event_id>:<start>")` → `"<tenant>:reminder:<gcal_event_id>:<start>"`: independiente del evento que la genera. Para barridos y handlers de calendario (`reminder:…`, `confirm:…`), donde distintos eventos pueden encontrar la misma entidad.
- El registro de conectores, en las operaciones que escriben, usa `side_effect` **sin transacción abierta durante la llamada**:

```
1. tenant_tx: buscar la clave en side_effect
     done     → devolver el resultado guardado, sin llamar al sistema externo
     failed   → el intento anterior no tuvo efecto: se puede volver a llamar
     unknown  → no se llama; se devuelve error y queda en Consola → Salud
     started  → updated_at hace < 2 min: otro intento puede estar en curso → TransientConnectorError (reintentar luego)
                más antiguo: el intento se cortó a mitad → según el manifest de la operación (§22.3):
                  on_interrupted=retry   → se vuelve a llamar (el destino es idempotente o un duplicado es aceptable)
                  on_interrupted=unknown → status=unknown y aviso
     no está  → INSERT status=started
   COMMIT                                   ← el "started" queda guardado ANTES de llamar
2. llamar al sistema externo (sin transacción)
3. tenant_tx: UPDATE status=done, result=…  (o failed si el sistema respondió que no lo hizo)  COMMIT
```

- Valores actuales: `booking.create_booking` → `retry` (id de evento determinista, §18.5); `booking.cancel_booking` → `retry` (cancelar dos veces es inocuo); `email.send` → `retry` (un email duplicado al gimnasio es aceptable).
- Los `unknown` aparecen en **Consola → Salud** con los botones "Marcar hecho" y "Reintentar".

#### 18.5 Reservas concurrentes (sin dobles reservas)

El candado de §18.1 es **por conversación**: dos contactos distintos pueden intentar coger el mismo hueco a la vez, y Google Calendar no impide solapes. **[DECIDIDO]**:

- `create_booking` del adaptador `google_calendar` (§22.2) toma un **advisory lock de Postgres** por `(tenant, calendario, día)` alrededor de "volver a comprobar el hueco → crear la cita":
  - lock de sesión con `pg_try_advisory_lock` en bucle (clave = hash de 64 bits de los tres valores), en una conexión sin transacción abierta, liberado en `finally` (si el proceso muere, se libera al cerrarse la conexión);
  - espera máxima ~10 s; si no lo consigue → `TransientConnectorError` y el trabajo se reintenta.
- **Id de evento determinista**: `base32hex(sha256(clave idempotente))[:32]` en minúsculas (Google exige base32hex, de 5 a 1024 caracteres).
- Dentro del lock:
  1. `events.get(id determinista)`: si existe y no está cancelado → **éxito** (es un reintento de una reserva ya hecha).
  2. Leer los eventos de ese día directamente de Google y comprobar solo que `[start, start + duration_min)` no se solapa con ninguno → si se solapa → `SlotTaken`.
  3. `events.insert` con el id determinista; si Google responde **409** (ya existe) → `events.get` y devolverlo.
  4. **Comprobar el resultado** de la escritura.
- `SlotTaken` lo recoge el bloque `confirm_booking`: "Ese hueco se acaba de ocupar" + **vuelve a ofrecer las horas libres de ese día** (solo las del día, partidas en mañana/tarde si no caben, §19.7).
- No protege contra una cita que el dueño crea a mano en Calendar en ese mismo segundo: aceptado (el dueño la ve y decide).

#### 18.6 Tareas programadas y barridos

- **Programar** (acción `ScheduleTask(key, task_name, run_at, payload, conversation_id=None, max_delay_h=6)`): upsert en `scheduled_task` por `(tenant, key)` + trabajo de Procrastinate con `schedule_at=run_at`. Si `conversation_id` no es NULL, el trabajo toma `conv:<h>` calculado desde la conversación; si no, `sched:<key>`. Reprogramar con la misma clave cancela el trabajo anterior y crea uno nuevo.
- **Cancelar** (`CancelTask(key)`): marca `cancelled` y borra el trabajo pendiente. Si no existe, no falla.
- **Al ejecutarse**: comprueba que `scheduled_task.status == scheduled` (si se canceló justo antes, no hace nada), **solo** crea su `inbound_event` (`source=system`, §17) y encola `process_event` en `inbound`; el handler `task:<name>` se ejecuta siempre dentro de `process_event` (vía `dispatch`). **El handler verifica que la entidad sigue siendo válida** antes de actuar.
- **Periódicas**: cada `Periodic` del pack se registra como tarea periódica de Procrastinate de plataforma, que **reparte**: obtiene los tenants activos con ese pack (`fp_active_tenants`, §18.2) y, por tenant, **solo** crea el evento `system` y encola `process_event` (nunca un trabajo gigante para todos). El reparto corre en UTC y calcula, por tenant, si le toca según `tenant.timezone`: los `cron` de `Periodic` se interpretan en la hora local del tenant.
- **Barridos [DECIDIDO]** (periódicas idempotentes sobre datos externos): cuando lo que hay que hacer depende de datos que puede cambiar otra persona (el dueño crea o mueve citas en Calendar), **no se programa una tarea por entidad**: un barrido periódico por tenant mira la ventana relevante y actúa sobre lo que encuentre, con una clave estable por entidad (`ctx.idem.stable`, §18.4). Es el caso de los **recordatorios de citas** (§19.7): cada 15 min, citas que empiezan dentro de 23–25 h, clave `reminder:<gcal_event_id>:<start>` (si la cita se mueve, cambia `start` y le corresponde un recordatorio nuevo; si no, el barrido la vuelve a ver y no repite).
- Recuperación: si el worker estaba caído a la hora programada, el trabajo se ejecuta al volver (sigue en la cola). Las periódicas y tareas con más retraso que su `max_delay_h` (por defecto 6 h, configurable en `Periodic` y en `ScheduleTask`) se descartan y se registran.

#### 18.7 Concurrencia del dueño y del bot

Si una conversación está en modo `human`, el motor no responde (solo guarda y reenvía al dueño). `human_until` (por defecto 12 h) devuelve la conversación al bot: se comprueba de forma perezosa en el siguiente `process_event`, sin trabajo propio. "Devolver al bot" lo hace al momento. Ese botón y la respuesta del dueño los procesa `process_owner_update` con el candado `conv:<h>` y respetando `conversation.version` (§17).

---

### 19. Packs: cómo conviven los distintos negocios

> **En pocas palabras.** Cada tipo de negocio es un **pack**: una carpeta de Python con su lógica. Los packs se construyen con **bloques** reutilizables (piezas como "elegir día" o "confirmar reserva") y definen sus **recorridos** en Python. Cada negocio concreto se diferencia por su **config** (precios, horarios, textos, opciones on/off), que se cambia en la consola al momento, y, si el pack lo necesita, por sus **datasets** (p. ej. los horarios de autobús). Añadir un negocio nuevo del mismo tipo no requiere programar; añadir un tipo de negocio nuevo es añadir una carpeta, sin tocar la plataforma.

#### 19.1 Contrato de un pack

```python
# packs/salon/pack.py
from core.packs import Pack, Periodic
from .config import SalonConfig           # esquema de config (pydantic)
from . import handlers, journeys

pack = Pack(
    name="salon",
    version="1.0.0",
    title="Peluquería / estética",
    config_schema=SalonConfig,                       # valida la config de negocio
    journeys=journeys.JOURNEYS,                      # recorridos en Python (§19.3)
    conversation_timeout_min=30,                     # inactividad que cierra la conversación (§19.6)
    handlers={                                       # qué hace con cada disparador
        "message": "conversation",                   # el motor de conversación con JOURNEYS
        "calendar": handlers.on_calendar_change,     # cita creada a mano por el dueño → confirmación
        "payload:reminder_confirm": handlers.confirm_from_reminder, # botones de la plantilla de recordatorio
        "payload:reminder_cancel": handlers.cancel_from_reminder,
    },
    periodic=[
        Periodic("reminders", every="15m", handler=handlers.send_reminders),      # barrido (§18.6)
        Periodic("daily_summary", cron="0 8 * * *", handler=handlers.daily_summary,  # 8:00 hora del tenant
                 max_delay_h=6),                    # más tarde, se descarta (§18.6)
    ],
    connectors={"booking": "required", "llm": "optional"},
    datasets={},                                     # este pack no usa datasets
    store=None,                                      # ni datos propios (§4.2)
    whatsapp_templates="templates/whatsapp.yaml",    # plantillas que este pack necesita
    knowledge_template="knowledge/template.md",
    scenarios="scenarios/",                          # tests de conversación (§25.2)
)
```

**Descubrimiento:** al arrancar, `flowpilot/packs_registry` importa `packs/*/pack.py` (importlib) y registra cada `pack`; `core` solo define los contratos. Añadir un pack = añadir una carpeta. **No se toca la plataforma.**

Todo handler tiene la firma `handler(event, ctx: PackContext) -> HandleResult`, con los eventos de dominio de §17.

**Tipos de handler:**

| Disparador | Clave del handler | Ejemplo |
|---|---|---|
| Mensaje de un contacto | `message` | Conversación de reserva |
| Botón con payload `<prefijo>:<valor>` (p. ej. en una plantilla) | `payload:<prefijo>` | "Cancelar" en el recordatorio → `payload:reminder_cancel` |
| Formulario recibido | `form` | Gimnasio: alta de un socio (§21.5) |
| Cambio en el calendario del tenant (tras sincronizar) | `calendar` | Cita creada a mano → confirmación al contacto |
| Tarea programada que vence | `task:<nombre>` | Seguimiento de una conversación |
| Tarea periódica | en `periodic` | Barrido de recordatorios; resumen diario al dueño |

[FUTURO] Handler `webhook:<nombre>` para avisos del software de un cliente (mismo patrón: ruta firmada → `inbound_event` → handler). Comandos de pack en el bot del dueño (`owner:<comando>`, p. ej. `/bloquear`, `/vacaciones`). Hoy el bot del dueño solo tiene los comandos genéricos de plataforma (§21.6).

#### 19.2 Config de negocio

Cada pack define su esquema con **pydantic**. El Admin y la consola validan con él antes de guardar.

```python
# packs/salon/config.py
class Service(BaseModel):
    id: str
    name: str = Field(max_length=24)        # cabe en una fila de lista de WhatsApp
    price_eur: Decimal | None = None
    duration_min: int                       # tiempo en que el profesional está ocupado (colisiones)
    client_presence_min: int                # tiempo del contacto en el local (última hora ofrecible)

class SalonConfig(BaseModel):
    business_name: str
    services: list[Service]                 # (la zona horaria es tenant.timezone, no va en la config)
    opening_hours: WeeklyHours              # {tue: ["10:00-14:00","16:00-21:00"], ...}
    slot_step_min: int = 30
    booking_horizon_days: int = 14
    min_notice_min: int = 60
    reminders_enabled: bool = True
    reminder_window_hours: tuple[int, int] = (23, 25)
    manual_booking_confirmation: bool = True
    texts: SalonTexts = SalonTexts()        # todos los textos con valores por defecto, editables
```

- Se guarda en `tenant_pack_assignment.config` (JSON) con **historial de versiones** (`config_history`) y quién lo cambió (auditoría).
- **Efecto inmediato** (sin despliegue), incluso en conversaciones en curso.
- `texts.ai_disclosure` no puede estar vacío si el tenant **usa IA** (lo exige la validación de config, §20.4).
- **Cambios del esquema [DECIDIDO]:** por defecto, **aditivos** (campos nuevos con valor por defecto), así las configs guardadas siguen validando. Un cambio incompatible exige que el pack defina `migrate_config(old: dict) -> dict`; `sync_packs` lo ejecuta en la tarea `migrate` sobre cada tenant y deja la versión nueva en `config_history`. `validate_packs` valida en CI las configs de ejemplo del pack.

#### 19.3 Motor de conversación

> **En pocas palabras.** Una conversación es una sucesión de **preguntas y respuestas**. El motor sabe en qué bloque y en qué paso está cada conversación, pasa el mensaje al bloque actual y, cuando el bloque termina, pasa al siguiente del recorrido. También se encarga de lo común a todas: "MENU", "HUMANO", "CANCELAR", preguntas libres a la IA y conversaciones abandonadas.

**Recorridos en Python [DECIDIDO]** (sin YAML de recetas, sin herencia entre recorridos):

```python
# packs/salon/journeys.py
from core.packs import Journeys, Step
from core.blocks import menu, ask_text, faq, handoff
from core.blocks.booking import (choose_service, choose_day, choose_slot, confirm_booking,
                                 list_my_bookings, cancel_booking, reschedule_booking)

JOURNEYS = Journeys(
    entry="main_menu",
    journeys={
        "main_menu":   [Step("menu", menu,
                              options=["book", "my_bookings", "reschedule", "cancel", "info", "human"],
                              keywords={"book": ["cita", "reservar", "pedir hora"],
                                        "cancel": ["cancelar", "anular"]})],
        "book": [
            Step("service", choose_service),
            Step("day",     choose_day, lookahead_days=lambda cfg: cfg.booking_horizon_days),
            Step("slot",    choose_slot),
            Step("name",    ask_text, validate="name", skip_if_known=True),
            Step("confirm", confirm_booking),
        ],
        "my_bookings": [Step("list", list_my_bookings)],
        "cancel":      [Step("cancel", cancel_booking)],
        "reschedule":  [Step("reschedule", reschedule_booking)],
        "info":        [Step("faq", faq)],
        "human":       [Step("handoff", handoff, reason="user_request")],
    },
)
```

- Un paso puede depender de la config (`when=lambda cfg: cfg.x`) o tomar parámetros de ella (como `lookahead_days`): así varía el flujo entre tenants **sin código por tenant**. `when` es una **función pura de la config**, se evalúa al entrar al paso, y un paso saltado no deja nada en `data`.
- `menu` acepta `keywords` deterministas por opción: "quiero cita" arranca la reserva directamente, sin pasar por la FAQ con IA.
- **Validación** (en CI y al arrancar): los bloques existen, los parámetros son válidos según `Params` del bloque, el recorrido de entrada existe y todo recorrido es alcanzable desde el menú (la alcanzabilidad ignora `when`).

**El contexto que recibe un pack:**

```python
class PackContext:
    tenant: TenantInfo              # id, nombre, zona horaria, idioma
    config: BaseModel               # la config validada (p. ej. SalonConfig)
    contact: ContactInfo | None     # id, nombre conocido, canal
    conversation: ConversationView | None   # estado actual (lectura); se modifica vía resultado
    connectors: Connectors          # ctx.connectors.booking.get_slots(...)  (§22)
    knowledge: Knowledge            # documento + ayudante de FAQ
    datasets: Datasets              # ctx.datasets.get("timetable") → objeto en memoria (§19.5)
    store: object | None            # protocolo de datos propios del pack, si lo declara (§4.2)
    clock: Clock                    # "ahora" inyectable (tests deterministas)
    channel: ChannelCapabilities | None     # qué soporta el canal (botones, listas, plantillas…)
    idem: IdempotencyHelper         # .key(name) ligada al evento; .stable(name) para barridos (§18.4)
    log: Logger
```

**Lo que devuelve:**

```python
class HandleResult:
    replies: list[Reply]            # Text, Buttons, List, Template, Media, Location
    actions: list[Action]           # ScheduleTask, CancelTask, Handoff, NotifyOwner, SendTemplate, UpdateContact, EmitMetric
                                    # UpdateContact(display_name=None, opt_out=None): la usan BAJA y ask_text con skip_if_known
    state: ConversationState | None # nuevo estado (None = sin cambios)
```

**Un bloque** (el motor y `StepResult` viven en `core/conversation/`; `Step` es solo la declaración de un paso dentro de un recorrido):

```python
class Block(Protocol):
    name: str                        # "choose_slot"
    version: int                     # se incrementa si cambia el formato de su estado
    Params: type[BaseModel]          # parámetros que acepta en un Step

    def start(self, ctx, params, data) -> StepResult: ...
    def on_input(self, ctx, params, data, block_state, msg) -> StepResult: ...

# StepResult: lo que devuelve un bloque
Ask(replies, expect=Expect.choice(options) | Expect.text() …, block_state={...})
Done(result=...)        # el bloque terminó; su resultado se guarda en data[<id del paso>]
Back()                  # volver al paso anterior
Abort(reason)           # cancelar el recorrido → volver al menú
Handoff(reason)         # derivar a humano
NotUnderstood()         # el bloque no entiende el mensaje (texto libre)
```

**Derivación a humano:** el bloque `handoff` (o el comando HUMANO) devuelve `StepResult.Handoff` → el motor lo traduce a la acción `Handoff` → `apply_actions` crea la fila `handoff`, pone `mode=human` y `human_until`, y emite el `NotifyOwner` de derivación (§21.6).

**Cómo el motor ejecuta un recorrido:**

```
mensaje → ¿payload cuyo prefijo coincide EXACTAMENTE con una clave
           payload:<prefijo> del pack?                               → handlers["payload:<prefijo>"] (vía dispatch)
           (si no coincide, p. ej. "slot_17:00", va al bloque actual como cualquier payload)
        → ¿comando global? (MENU / HUMANO / CANCELAR / BAJA)        → lo resuelve el motor
                                     (BAJA → UpdateContact(opt_out=True): sin plantillas ni recordatorios)
        → ¿conversación en modo humano?                              → no responde; reenvía al dueño
        → ¿conversación caducada (inactiva > conversation_timeout_min)? → se cierra y se abre otra (recorrido de entrada)
        → ¿el bloque guardado (nombre + versión) ya no existe?       → reinicia con aviso amable (§19.6)
        → bloque actual.on_input(msg)
             ├─ Ask   → responder y guardar block_state
             ├─ Done  → guardar resultado → siguiente bloque.start()  (o fin del recorrido)
             └─ NotUnderstood (texto libre que el bloque no entiende)
                   → ¿el tenant usa IA? → FAQ: responder con el knowledge y REPETIR la pregunta actual
                   → si no → mensaje de "no te he entendido" + repetir la pregunta
```

**Estado de una conversación** (JSON en `conversation.state`):

```json
{
  "journey": "book",
  "step_id": "day",
  "block": {"name": "choose_day", "version": 1, "state": {"page": 0}},
  "data": {"service": {"id": "corte", "duration_min": 30, "client_presence_min": 30}},
  "started_at": "2026-10-03T10:12:00+02:00"
}
```

`step_id` es la única referencia a la posición: si el recorrido cambia y ese paso ya no existe, se reinicia como en §19.6. Se añade `"real_connectors": true` solo en conversaciones del simulador que el operador pasa a conectores reales (§21.4).

#### 19.4 Bloques

**Bloques comunes** (en `core/blocks/`, los usa cualquier pack):

| Bloque | Qué hace | Parámetros típicos |
|---|---|---|
| `menu` | Muestra opciones y salta al recorrido elegido; reconoce palabras clave por opción | `options`, `keywords`, `text_key` |
| `choose_option` | Elegir de una lista (estática o de la config), paginada si no cabe | `source`, `text_key` |
| `ask_text` | Pedir un texto con validación | `validate` (`name`, `email`, `regex`), `skip_if_known` |
| `ask_date` | Pedir una fecha con botones ("Hoy", "Mañana", "Otro día") o texto, interpretado de forma determinista ("el jueves", "12/10") | `min`, `max`, `allow_free_text` |
| `confirm` | Resumen + Sí/No | `summary_template` |
| `show_info` | Enviar un texto o documento y terminar | `text_key`, `media` |
| `handoff` | Derivar a humano con motivo | `reason` |
| `faq` | Recorrido de pregunta libre con la IA (§20) | `max_turns` |

[FUTURO] Extracción de fechas con IA en `ask_date`, cuando el texto libre determinista se quede corto.

**Bloques de reservas** (en `core/blocks/booking/`, para peluquería, clínica, talleres…; usan el conector `booking`):

| Bloque | Qué hace |
|---|---|
| `choose_service` | Lista de servicios desde la config |
| `choose_day` | Días con hueco (`booking.get_available_days`) |
| `choose_slot` | Horas libres del día (`booking.get_slots`); si no caben en una lista, pregunta antes mañana/tarde |
| `confirm_booking` | Confirma y **crea la cita** de forma idempotente; si el hueco se acaba de ocupar (`SlotTaken`), vuelve a ofrecer las horas de ese día (§18.5) |
| `list_my_bookings` | Citas futuras del contacto |
| `cancel_booking` | Elegir cita → confirmar → cancelar |
| `reschedule_booking` | Cancelar + reservar, en un solo recorrido |

Los bloques construyen `ServiceSpec` (duración y presencia del servicio elegido) y `AvailabilityRules` (horario semanal, paso, antelación mínima, zona horaria) **desde `ctx.config`** y se los pasan al conector (§22.2).

[FUTURO] `join_waitlist` y la lista de espera; `choose_staff` (elegir profesional) cuando haya negocios con varios profesionales.

**Bloques propios de un pack** (p. ej. `packs/bus/blocks/ask_locality.py`): mismo contrato, solo visibles en ese pack.

#### 19.5 Datasets

> **En pocas palabras.** Algunos negocios funcionan con **datos estructurados** que no son una config sencilla: los horarios de una empresa de autobuses, por ejemplo. Un pack los declara como **dataset**: un formato propio, un lector y un **validador estricto que no deja pasar nada ambiguo**. El operador los sube desde la consola; si validan, se guardan como una versión nueva y se activan. El bot los consulta en memoria, sin ir a la base de datos en cada pregunta.

**Contrato:**

```python
# core/datasets
class DatasetSpec(Protocol):
    name: str                                      # "timetable"
    def parse(self, raw: bytes) -> dict            # fichero canónico (YAML/JSON) → dict canónico
    def validate(self, content: dict, tenant: TenantInfo) -> list[DatasetError]   # vacío = válido
    def load(self, content: dict) -> object        # dict → objetos inmutables e índices para consultar

@dataclass
class DatasetError:
    file: str; row: str | None; reason: str        # error concreto: fichero, fila, motivo
```

```python
# packs/bus/pack.py (extracto)
datasets={"timetable": TimetableDataset()}
```

**Ciclo de vida [DECIDIDO]:**

1. **Conversión** (una vez por cliente, fuera de la plataforma): una herramienta en `tools/` convierte el Excel u otro formato del cliente al formato canónico del pack. No forma parte del runtime.
2. **Subida** (Consola → Subir dataset): `parse` + `validate` en la propia petición (los datasets son pequeños: **máximo 5 MB**, y la validación debe tardar **menos de 10 s**). Con errores → informe y **no se guarda** (métrica `dataset_upload{result=rejected}`). Sin errores → nueva fila en `dataset_version` (`is_active=false`, `checksum`, `uploaded_by`).
3. **Activación**: el operador activa la versión (una sola activa por tenant y dataset). Al activar se **vuelven a ejecutar `validate` + `load` con el código actual**: solo se activa si pasa. Queda en `audit_event`.
4. **Uso**: tx1 del runtime (§17) lee el id de la versión activa; el worker mantiene una **caché en memoria por `(tenant, name, versión)`** con el resultado de `load`. Las consultas del pack son en memoria, sin E/S. Una versión nueva entra sola en el siguiente evento.
5. **Si el worker no puede cargar la versión activa** (p. ej. un cambio de código que la vuelve incompatible): el pack responde con un mensaje seguro ("ahora no puedo consultar los horarios") y salta la alarma 🔴 (§12.4). Los tests del pack incluyen datasets de ejemplo para detectarlo en CI.

[FUTURO] Vista de revisión y diff entre versiones para el negocio (HTML/PDF), para que el cliente confirme sus datos antes de activarlos.

#### 19.6 Versiones y conversaciones en curso

- La conversación guarda la **versión del pack** en la columna `conversation.pack_version` (formato `"1.0.0"`, la de `Pack.version`) con la que empezó, para trazas y depuración. Siempre se ejecuta con el código desplegado.
- El estado guarda **nombre y versión del bloque** actual. Si tras un despliegue ese bloque (con esa versión) ya no existe, el motor **reinicia con amabilidad** ("Perdona, he tenido que reiniciar; ¿qué querías hacer?") y lo registra.
- Cambio de bloque **compatible** (mejora interna, parámetro opcional nuevo): se modifica el bloque; los escenarios de todos los packs garantizan que nada se rompe. Cambio **incompatible** del formato del estado: se sube `version` (las conversaciones a mitad de ese bloque se reinician como arriba).
- Hay **una sola conversación abierta** por `(channel_id, contact_id)`. Si lleva inactiva más de `conversation_timeout_min` (manifest del pack, por defecto 30 min), en el siguiente mensaje se cierra (`closed_at`, de forma perezosa) y se abre otra; el aviso de IA (§20.4) se envía al abrir cada conversación nueva.
- Los cambios de **config** y de **datasets** se aplican al momento, también en conversaciones en curso.

**[FUTURO] Recetas y variantes por tenant.** Cuando varios tenants de un mismo pack necesiten flujos distintos que la config no cubra:

- `tenants/<slug>/recipe.yaml`: qué recorridos y bloques usa ese tenant (sin lógica), validado igual que los recorridos en Python, con la versión de la receta guardada en la conversación (junto a `pack_version`).
- `tenants/<slug>/variant.py`: sustituye un handler concreto, solo como último recurso.
- **Regla de los tres:** si la misma variación aparece en tres tenants, se convierte en parámetro de bloque, en opción de config o en bloque del pack.
- Editor de recetas en la consola, con ejecución de escenarios antes de activar.

Encaja sin romper nada: `Journeys` es ya el formato al que se convertiría una receta.

#### 19.7 Packs iniciales

**`salon` (peluquería / estética / barbería)**

- **Google Calendar es la fuente de verdad**: el dueño gestiona su agenda desde Calendar y el bot lee y escribe en él (conector `booking`, §22.2).
  - **Citas manuales**: si el dueño crea una cita con `Telefono: +34…` en la descripción, el handler `calendar` (tras la sincronización; solo eventos creados después del alta del watch, §22.2) devuelve `SendTemplate(to=ContactRef(phone=…), template="booking_confirmation", …)` (§21.1), que sale por el canal WhatsApp principal del tenant, con clave estable `confirm:<gcal_event_id>` (si `manual_booking_confirmation` está activo). Si no hay teléfono o el tenant no tiene canal WhatsApp principal, se registra y no se envía.
  - **Eventos de configuración** en el calendario: `[CFG] CERRADO` (día cerrado), `[CFG] VACACIONES` (todo el rango del evento cerrado), `[CFG] HORARIO HH:MM-HH:MM` (horario especial ese día). **Prioridad:** cerrado/vacaciones > horario especial > horario semanal de la config.
- **Huecos**: función pura `compute_slots(day, service: ServiceSpec, rules: AvailabilityRules, busy, closures, now)` en `core/blocks/booking/slots.py` → horas libres. Usa `duration_min` para las colisiones con otros eventos y `client_presence_min` para decidir la última hora ofrecible (el contacto debe poder terminar antes del cierre: unas mechas de 60 min de profesional y 180 min de presencia no se ofrecen a las 19:00 si se cierra a las 21:00). Respeta la antelación mínima y el paso. La llama el adaptador `google_calendar` con los eventos y cierres que lee (§22.2).
- Si el día tiene más horas de las que caben en una lista, **se parte en mañana/tarde** (botones) antes de mostrar la lista.
- **Recuperación de `slot_taken`** (§18.5): vuelve a ofrecer solo las horas libres de ese día.
- Recorridos: menú, reservar, mis citas, cancelar, cambiar, información (FAQ con IA), humano.
- **FAQ con IA** (§20): responde solo desde el `knowledge` del tenant vía Bedrock; si no está, respuesta segura. Incluye el aviso de IA (§20.4).
- **Recordatorios por barrido** (§18.6): cada 15 min, citas que empiezan dentro de `reminder_window_hours` (23–25 h) → `SendTemplate` con la plantilla de recordatorio (botones Confirmar/Cancelar con payloads `reminder_confirm:<gcal_event_id>` y `reminder_cancel:<gcal_event_id>`, §19.3), una sola vez por clave estable `reminder:<gcal_event_id>:<start>`. Cubre también las citas que crea o mueve el dueño. Destinatario: en citas del bot, el contacto guardado en `extendedProperties.private.fp_contact_id`, **por su canal original** (en Telegram, el cuerpo de la plantilla de `templates/whatsapp.yaml` del pack, con sus parámetros, como mensaje normal); en citas manuales, el teléfono de la descripción por el canal WhatsApp principal; contactos con `opt_out` no reciben nada.
- Periódicas: `reminders` (15 min), `daily_summary` al dueño (8:00) vía `NotifyOwner`.
- Conectores: `booking` (obligatorio), `llm` (opcional).
- [FUTURO] Comandos `/bloquear` y `/vacaciones` por Telegram; `review_request`; `waitlist_offer`; `reactivation`.

**Detalles de Google Calendar a respetar** (los cumple el adaptador, §22.2):

- En eventos de día completo, `end.date` es **exclusivo** (un evento del 20 al 22 tiene `end.date` = 23): iterar con `d < end`.
- Calendar puede **inyectar HTML** (`<br>`, `<p>`, entidades) en las descripciones editadas a mano: limpiar antes de parsear `Telefono:` y demás campos (tolerante a mayúsculas, tildes y espacios).
- **Lecturas por ventana** (`get_slots`, `list_bookings`, cierres): `singleEvents=true` (expande recurrentes) + `timeMin`/`timeMax` + **paginación** (`nextPageToken`) con un tope de páginas.
- **Sincronización incremental** (`sync_changes`): **sin** `singleEvents`, **sin** tope de páginas y **sin** filtro de fechas: `nextSyncToken` solo llega en la última página, y las peticiones con `syncToken` no admiten `timeMin`/`timeMax` ni `privateExtendedProperty`.
- Alcance mínimo: `calendar.events`.
- **Comprobar siempre el resultado de una escritura** (lo devuelto por la API, no suponer éxito).
- **Id de evento determinista**: `base32hex(sha256(clave idempotente))[:32]` en minúsculas (§18.5). Los eventos del bot guardan en `extendedProperties.private` `fp_contact_id` y `fp_key` (clave idempotente): así se distinguen de los manuales y se sabe a quién y por qué canal avisar. `list_bookings_for_contact` filtra con `privateExtendedProperty=fp_contact_id=<id>`.
- Sin caché de huecos: siempre se lee de Google (el volumen previsto lo permite).
- Zonas horarias siempre con `datetime` con zona (`tenant.timezone`, `Europe/Madrid` por defecto); convertir lo que devuelve la API.

**`bus` (horarios de autobús)**

- **Datos**: un dataset `timetable` (§19.5) por tenant, en formato canónico:
  - **localidades** (lo que elige el usuario: id, nombre, alias) y **paradas** físicas (código, nombre público, localidad);
  - **observaciones** con ámbito (de viaje o de parada; condición o aviso) y el texto exacto que ve el contacto;
  - **líneas** con sus **temporadas** (rangos día/mes que cubren el año sin huecos ni solapes) y **tablas** por temporada y clase de día (cabecera ordenada de paradas, viajes con horas y observaciones, marca de "mismo autobús");
  - **calendario del tenant**: festivos (con ámbito por localidad o línea) y periodo escolar.
- **Validador estricto**: parada desconocida, observación sin definir, horas que retroceden, día de la semana sin declarar, temporada que no cubre el año… → error con fichero, fila y motivo. Un horario mal cargado hace que alguien pierda un autobús.
- **Motor de consulta en memoria** (origen → destino → fecha), sin E/S:
  - salidas **directas**: el viaje pasa por una parada del origen y, más adelante en su orden, por una del destino;
  - **fusión del mismo autobús** (mismas horas del par en líneas distintas → una sola salida con las notas de ambas);
  - si la fecha es hoy, retira o marca las **salidas ya pasadas**;
  - **`sin_datos` ≠ sin servicio**: si la fecha no está cubierta por los datos, lo dice y da el teléfono; nunca lo presenta como "no hay autobús";
  - **nunca inventa trasbordos**: sin trayecto directo → lo dice y muestra los destinos que sí hay.
- **Matcher de localidades determinista**: normalización (minúsculas, sin tildes ni puntuación, sin artículos ni `de`/`del`, sin relleno como "desde", "a", "para"), alias, prefijo único, erratas con Damerau-Levenshtein (≤ 1 hasta 5 letras, ≤ 2 si es más larga; mínimo 3 letras). **Nunca adivina en silencio**: coincidencia única → continúa (una errata única se confirma con Sí/No); varias → botones o lista para elegir; ninguna → "No conozco ese pueblo" + "Ver pueblos por línea".
- **"Ver pueblos por línea"**: lista de líneas y luego de pueblos, **paginada** (8 por página + "Más" + "Volver" = 10 filas).
- Recorrido: menú → origen (`ask_locality`) → destino → día (`ask_date`) → resultado.
- **Sin IA. Sin tablas relacionales propias** (todo vive en el dataset).
- Registra los **textos no entendidos sin el teléfono** (texto normalizado + paso), para descubrir alias que faltan.
- [FUTURO] Incidencias (avisos activos por línea); exportación GTFS (un script sobre el modelo en memoria).

**`gym` (alta de socios)**

- **Formulario de alta** alojado por la plataforma (§21.5): página pública por tenant con los campos que define la config del pack (`form_fields`: nombre, tipo, obligatorio) + consentimiento RGPD. **Sin datos de salud** en el formulario.
- Handler `form`: valida contra la config → envía un **email con los datos al Gmail del gimnasio** (`config.notify_email`) vía conector `email` (SES), idempotente con `ctx.idem.key("form_email")`.
- Nada al socio por ahora (la página muestra un texto de gracias de la config).
- Conectores: `email` (obligatorio). No necesita canal de mensajería.
- [FUTURO] Resumen semanal al gimnasio.

#### 19.8 Cómo se añade un pack nuevo (checklist)

1. `packs/<nombre>/pack.py` con el manifest.
2. `config.py` con el esquema y valores por defecto.
3. `journeys.py` (si es conversacional) y bloques propios si hacen falta.
4. Handlers de formularios, calendario, payloads o tareas.
5. Datasets (parser + validador + datos de ejemplo) si los necesita; `storage.py` solo si de verdad necesita tablas propias (§4.2).
6. Plantillas de WhatsApp y plantilla de knowledge, si aplican.
7. **Escenarios de test** (mínimo: camino feliz de cada recorrido + 3 caminos de error).
8. Entrada en `docs/packs/<nombre>.md` (qué hace, qué config pide, qué conectores y datasets necesita).
9. PR → CI → revisión → despliegue → alta del tenant desde la consola.

---

### 20. Inteligencia artificial

> **En pocas palabras.** La IA es **de apoyo**: hoy solo contesta preguntas sobre el negocio usando su documento de información (knowledge) en el pack de peluquería. Va por **Amazon Bedrock** con el tráfico dentro de la UE. **Nunca** decide reservas, precios ni horarios, y nunca charla de temas ajenos al negocio (Meta lo prohíbe en WhatsApp). Los usos se irán añadiendo uno a uno, cuando aporten.

#### 20.1 Interfaz y usos

```python
class LLMConnector(Protocol):
    def complete(self, system: str, messages: list[ChatMessage],
                 schema: type[BaseModel] | None = None) -> Completion
# Completion(text: str, data: BaseModel | None, tokens_in: int, tokens_out: int)
```

- Una sola operación genérica: con `schema`, la salida se pide estructurada y se valida con pydantic (`data`); sin él, texto (`text`).
- Adaptadores: `bedrock` (perfil de inferencia geográfico **EU**: el tráfico se queda en regiones de la UE) y `mock`. Autenticación con el **rol IAM de la tarea**, sin claves. El id de modelo va por configuración (`LLM_DEFAULT_MODEL` o el `connector_binding` del tenant), **nunca en el código**.
- **Uso actual:** FAQ con knowledge en `salon` (§19.7). Responde solo desde el knowledge; si no está → respuesta segura ("No tengo esa información; te paso con el equipo") y se registra la pregunta sin respuesta.
- [FUTURO] Otros usos (transcribir audios, extraer fechas escritas a mano, clasificar urgencia) se añaden con la misma interfaz cuando un pack lo necesite.

#### 20.2 Principios y guardrails

- **La IA no decide** precios, horarios ni reservas: esos datos salen de la config, del calendario o de un dataset.
- System prompt fijo por plataforma + datos del negocio: "eres el asistente de <negocio>; responde solo sobre <negocio> con la información proporcionada; si no está, dilo".
- **Minimización:** sin teléfonos en los prompts; el nombre solo si hace falta; **nunca datos de salud**.
- Temperatura baja; historial corto.

#### 20.3 Cuotas y coste

- Cada llamada registra tokens y coste en `execution` y `usage_daily`.
- **Cuota mensual por tenant** (`tenant.llm_monthly_token_quota`); al superarla → respuesta segura y `ops_alert{kind=llm_quota}` (§12.4).
- [FUTURO] Evals por pack (preguntas con el comportamiento esperado) contra el modelo real al cambiar de modelo o de prompt.

#### 20.4 Cumplimiento

- **Aviso de IA** (AI Act art. 50, desde el 2/8/2026): obligatorio en las conversaciones donde interviene IA. "Usar IA" significa siempre que **el tenant tiene un binding `llm` activo**: solo entonces hay FAQ con IA, se muestra el aviso y se exige `ai_disclosure`. El motor inserta `texts.ai_disclosure` en el primer mensaje de cada conversación **nueva** de un tenant que usa IA ("Soy el asistente automático de X. Escribe HUMANO para hablar con una persona") y **no se puede desactivar**; la validación de config exige que `ai_disclosure` no esté vacío. [VERIFICAR alcance: si aplica también a bots sin IA, como `bus`].
- **Política de WhatsApp:** prohibido el chat de propósito general; el system prompt rechaza los temas ajenos al negocio.
- Proveedores de IA = subencargados de tratamiento (en el DPA).

---

### 21. Canales

> **En pocas palabras.** Cada canal (WhatsApp, Telegram…) tiene un **adaptador** que traduce sus mensajes a un formato común y viceversa. Los packs nunca saben por qué canal hablan: piden "botones" y el adaptador los convierte en lo que el canal soporte (botones, lista o texto numerado). Hoy los clientes de los negocios hablan por **WhatsApp y Telegram**; el email llegará más adelante. El cliente es dueño de sus cuentas.

#### 21.1 Contrato común

```python
@dataclass
class InternalMessage:                    # lo que reciben los packs
    id: str                               # id del proveedor (wamid, update_id…)
    tenant_id: UUID
    channel_id: UUID
    contact_external_id: str              # teléfono E.164, id de Telegram, id de sesión del simulador
    type: Literal["text","button","list","media","location","command","system"]   # button = botón interactivo o de plantilla
    text: str | None
    payload: str | None                   # id del botón o de la fila elegida
    media: MediaRef | None                # [FUTURO] fichero subido a S3; hoy solo el tipo
    received_at: datetime
    raw_ref: UUID                         # InboundEvent original

class ChannelAdapter(Protocol):
    type: str
    capabilities: ChannelCapabilities     # max_buttons, max_button_title, max_list_rows, max_text_len,
                                          # max_interactive_body, supports_templates, window_hours…
    def verify(self, request, secret_ref) -> bool                  # firma
    def parse(self, payload, channel) -> list[ParsedItem]          # mensajes + avisos de estado
    def render(self, replies, channel) -> list[OutboundPayload]    # Reply → formato del canal
    def send(self, outbound, channel) -> SendResult                # llamada a la API
```

`secret_ref` es de dónde sale el secreto para verificar: la `whatsapp_app` en WhatsApp (el webhook es por app) y el canal en Telegram.

**Degradación** (en `core/channels/`): lista → botones → texto numerado, según `capabilities`. Un usuario que contesta "2" a un texto numerado se traduce al `payload` de la opción 2.

**Plan de envío** (`plan_sends`), común a todos los canales:

1. **Fusionar** los textos consecutivos de un mismo turno en un solo mensaje.
2. Si un texto va seguido de botones o lista, **meter el texto en el cuerpo** del interactivo, salvo que texto + cuerpo superen `capabilities.max_interactive_body` (1.024 en WhatsApp): entonces el texto va en un mensaje aparte.
3. Aplicar los límites del canal (degradar o partir). Si algún título de botón supera `max_button_title` (20 en WhatsApp) → lista en vez de botones.
4. **Ventana de 24 h** (WhatsApp): si está cerrada y la respuesta no es una plantilla → usar `Reply.fallback_template` si el bloque o el pack la indica; si no, no enviar y emitir `ops_alert` (§12.4).
5. Crear un `SendIntent` por mensaje final, en orden.

**Envíos sin conversación abierta** (acción `SendTemplate(to: ContactRef(phone | contact_id), template, params)`): la usan barridos y handlers de calendario. La plataforma resuelve el destinatario: con `contact_id`, su canal y conversación; con `phone`, hace *upsert* de `contact` y `conversation` en el **canal WhatsApp principal** del tenant (`channel.is_default`). Si no hay canal resoluble, se registra y no se envía. Contactos con `opt_out` nunca reciben plantillas.

**Por qué la fusión es obligatoria:** desde el 1/10/2026 Meta cobra los mensajes de servicio por encima de 1.000 al mes por número (§21.2); menos mensajes = menos coste para el cliente.

#### 21.2 WhatsApp (Cloud API)

**Modo de conexión** (`whatsapp_app.app_mode`):

| Modo | Cuándo | Quién es dueño de la Meta App | Qué guardamos |
|---|---|---|---|
| `client_app` | **Ahora** | **El cliente** (en su Business Portfolio). El alta la hace el operador dentro de ese portfolio | App (`whatsapp_app`): `app_id`, `app_secret` (cifrado), `verify_token` (hash). Número (`whatsapp_connection`): `access_token` (cifrado), `waba_id`, `phone_number_id` |
| `platform_app` | [FUTURO] Tech Provider + Embedded Signup | Nosotros. El cliente conecta con Embedded Signup | Una sola app de plataforma; por número: `waba_id`, `phone_number_id`, token de negocio (cifrado) |

**Enrutado.** Meta configura el webhook **por app**, no por número: la URL identifica la app (`/webhooks/whatsapp/<app_public_id>`) para elegir el `app_secret` con el que verificar la firma, y el canal (y con él el tenant) se resuelve por `entry[].changes[].value.metadata.phone_number_id` con una función `SECURITY DEFINER` (§23.2), comprobando que el número pertenece a esa app. `parse`, `render` y `send` son iguales en los dos modos; en `platform_app` la diferencia es que una única URL recibe los números de todos los tenants y el enrutado depende solo del `phone_number_id`.

**Checklist de alta `client_app`** (la hace el operador con el cliente):

1. **SIM del negocio** con un número que **no tenga WhatsApp activo** (si lo tiene, hay que darlo de baja antes).
2. **Gmail del negocio** (cuenta a nombre del cliente).
3. **Meta Business Account** (Business Portfolio) del cliente, con el operador como administrador.
4. **App** de Meta en ese portfolio con el producto WhatsApp.
5. **Número registrado** en la WABA y verificado; **app suscrita a la WABA** (`POST /<WABA_ID>/subscribed_apps`) y campo `messages` suscrito en el webhook de la app.
6. **System User** con **token permanente** (el token temporal caduca en 24 h: nunca en producción).
7. **Plantillas** creadas **pronto**: tardan de 1 a 48 h en aprobarse.

**Entrada:**

- `GET /webhooks/whatsapp/<app_public_id>`: verificación inicial (`hub.verify_token` de la app, comparado con su hash).
- `POST /webhooks/whatsapp/<app_public_id>`:
  - verificar `X-Hub-Signature-256` (HMAC-SHA256 del cuerpo con el `app_secret` de la app), en tiempo constante;
  - resolver el canal de cada `change` por `metadata.phone_number_id` (desconocido → se registra y se ignora);
  - separar en elementos: mensajes (`text`, `interactive.button_reply`, `interactive.list_reply`, `button` —respuesta a un botón de plantilla, con `button.payload`—, `audio`, `image`, `document`, `location`) y avisos de estado (`sent`, `delivered`, `read`, `failed`, con su `pricing` y el `biz_opaque_callback_data` del envío, §18.3);
  - un `InboundEvent` por elemento (`provider_event_id` = `wamid` o `<wamid>:<status>`).
- Audios, imágenes, documentos y stickers: hoy se responde pidiendo que lo escriba o use los botones. [FUTURO] descarga en el worker a un bucket `media`.

**Salida:**

- `POST https://graph.facebook.com/<versión>/<phone_number_id>/messages`. La versión de la Graph API va en configuración (`META_GRAPH_VERSION`), nunca en el código.
- **Límites** (en `capabilities`): botones **≤ 3** (título ≤ 20 caracteres); lista **≤ 10 filas en total** sumando secciones (título ≤ 24, descripción ≤ 72); texto **≤ 4.096** caracteres.
- Tipos: `text`, `interactive` (`button`, `list`), `template`, `location`.
- Todo envío lleva `biz_opaque_callback_data = intent_id` (vuelve en los avisos de estado; sirve para conciliar los `unknown`, §18.3).
- Se guarda el `wamid` devuelto en `SendIntent` y en `Message`.
- Errores: 401 → token caducado o revocado (`ops_alert`, §12.4); otros 4xx → no reintentar; **no registrar el cuerpo de las respuestas 4xx** (puede contener datos sensibles).

**Plantillas** (`message_template`): por tenant y canal, con nombre, idioma, categoría (`utility`/`marketing`/`authentication`), componentes y estado (`pending`/`approved`/`rejected`). Los packs declaran las que necesitan (`templates/whatsapp.yaml`); en el alta se crean y se envían a aprobación vía API; la consola muestra su estado.

**Costes:** el aviso de estado trae `pricing.category` y `pricing.billable` → `message.pricing_category`, `message.billable`. El coste se calcula con `channel_rate` (país, categoría, precio, vigente desde) y se acumula en `usage_daily` como métrica de operación. Desde el 1/10/2026, Meta cobra los **mensajes de servicio** por mensaje al precio *utility* del país (página oficial de Meta, *Pricing for non-template messages*), con un tramo gratuito de **1.000 al mes por número** que recogen fuentes secundarias [VERIFICAR en la documentación oficial; fuentes en §12.4]; y las **plantillas *utility* enviadas dentro de la ventana de 24 h también se cobran** (https://www.courier.com/blog/whatsapp-pricing-changes-october-2026). Por eso: fusión de mensajes (§21.1), recordatorios y confirmaciones contados como coste, y **alerta** cuando un número pasa de 800 mensajes de servicio en el mes (§12.4).

[FUTURO] **WhatsApp Flows**: bloque que envía un Flow (p. ej. reserva en una sola pantalla) y recibe `nfm_reply`; en canales sin Flows se degrada a la secuencia de bloques normal.

#### 21.3 Telegram

- Un **bot por tenant** (creado con @BotFather a nombre del cliente; el token se guarda cifrado), si el tenant usa Telegram como canal con sus contactos.
- Alta (vía `console_action`): `setWebhook(url=…/webhooks/telegram/<channel_public_id>, secret_token=<aleatorio por canal>, allowed_updates=[message, callback_query])`. El `secret_token` se guarda como hash.
- Verificación: cabecera `X-Telegram-Bot-Api-Secret-Token` comparada (su hash) en tiempo constante.
- `provider_event_id` = `<channel_id>:<update_id>` (`update_id` solo es único por bot). Ojo: tras una semana sin updates, Telegram empieza el siguiente `update_id` en un valor aleatorio; la deduplicación solo es fiable dentro de la ventana de reintentos, que es lo que importa.
- Botones: `InlineKeyboardMarkup` → `callback_query.data` = `payload`; se responde siempre con `answerCallbackQuery`. Más botones que WhatsApp, sin ventana de 24 h, sin avisos de estado y gratis.
- [FUTURO] Mini Apps para formularios ricos; Business Mode.

#### 21.4 Canal de pruebas (`test`)

- **Simulador** en la consola (§8.2): el operador elige un tenant y conversa con su bot como si fuera un contacto. Muestra las respuestas renderizadas, el estado de la conversación, el bloque actual y las llamadas a conectores.
- Por dentro es un canal más (`channel.type = test`): el mensaje entra como `InboundEvent` (`source=test`) y sigue el camino normal (cola, worker, pack, `send_intent`). El "envío" solo guarda el mensaje; la página lo recoge con un refresco periódico de HTMX.
- `capabilities` elegibles (como WhatsApp o como Telegram) para probar la degradación.
- **Conectores**: en conversaciones de un canal `test`, el proxy de conectores sustituye **todos** los adaptadores por `mock`, salvo que `conversation.state.real_connectors = true` (lo fija el operador desde el simulador; queda en `audit_event`; las llamadas reales se hacen en el worker, nunca en `web`).
- **Acciones**: `NotifyOwner` se muestra en el simulador y **no se entrega** al dueño; `ScheduleTask` se ignora salvo en modo real.
- **Disponibilidad**: local y staging; en prod, **solo para el operador autenticado** (dentro de `consola.<dominio>`).
- Lo usa el **smoke test** del despliegue (§10.2) con el tenant interno `smoke`.

#### 21.5 Formularios públicos

- Rutas `GET/POST /f/<public_id>` servidas por `web` (en `api.<dominio>`). `public_id` es aleatorio y no revela el tenant; se resuelve con una función `SECURITY DEFINER` como los canales (§23.2).
- La página se genera a partir de los campos de la config del pack (`form_fields`) + **casilla de consentimiento RGPD** obligatoria (texto y enlace a la política de privacidad del negocio en la config).
- **Protecciones** (§5.3): CSRF, campo trampa (honeypot), rate limit por IP, tamaño máximo, sin ficheros, validación por esquema.
- Cada envío válido genera un `InboundEvent` con `source=form`; `provider_event_id` = id de envío de un solo uso incluido en la página (un doble clic no duplica). Lo procesa el handler `form` del pack como `FormSubmitted` (§17).
- Formulario en estado `paused` o tenant `offboarded` → página "formulario no disponible".

#### 21.6 Avisos al dueño y bot del dueño

> **En pocas palabras.** Los packs avisan al dueño con una acción `NotifyOwner` sin saber por dónde le llega. Cada tenant tiene un `owner_channel`; hoy es Telegram, con un único bot de la plataforma. Es gratis, rápido de construir y evita tener que hacer una web para clientes durante mucho tiempo.

- **`NotifyOwner(kind, text, data)`**: la plataforma lo entrega por el `owner_channel` del tenant (por defecto `telegram`); [FUTURO] WhatsApp o email del dueño, con el mismo contrato. Se guarda en `owner_notification` y se entrega por la cola `outbound` con clave idempotente `ctx.idem.key("notify:<kind>")` (guardada en `owner_notification.key`). Sin dueño vinculado → se registra y aparece en la consola.
- **Un solo bot de plataforma** (token en los secretos de plataforma). Webhook: `POST /webhooks/owner_bot`, verificado con `OWNER_BOT_WEBHOOK_SECRET` (cabecera `X-Telegram-Bot-Api-Secret-Token`) antes de nada → el usuario de Telegram se resuelve a sus `owner_link` con una función `SECURITY DEFINER` → se guarda `inbound_event(source=owner_bot)` y se encola `process_owner_update(event_id)` en `inbound` (con `conv:<h>` si la respuesta va a una conversación; si no, sin candado) → 200.
- **Vinculación:** la consola genera un enlace `https://t.me/<bot>?start=<código de un solo uso, caduca en 24 h>` (guardado como hash en `owner_link_code`) → al abrirlo se crea `owner_link(tenant_id, telegram_user_id, role)`. Un usuario de Telegram puede estar vinculado a varios tenants: `/negocio` elige el activo (`owner_link.active_tenant`).
- **Avisos**: derivación a humano, resumen diario, cancelaciones, errores de un conector del tenant, plantilla rechazada.
- **Derivación con respuesta:** el aviso de derivación incluye el hilo; el dueño **responde a ese mensaje en Telegram** → se asocia a la conversación por `reply_to_message.message_id` = `owner_notification.telegram_message_id` → se envía al contacto por su canal original (dentro de la ventana de 24 h; fuera, se avisa de que hace falta una plantilla), con el candado `conv:<h>` (§17). Botón "Devolver al bot" → la conversación vuelve a modo bot.
- **Hoy:** `/negocio`, responder a una derivación y "Devolver al bot". [FUTURO] `/pausar`, `/reanudar`, `/estado`, `/soporte <texto>` y comandos de pack (§19.1).
- **Alertas de operación (ops):** no usan `owner_link` ni salen de la aplicación: la aplicación emite la métrica `ops_alert{kind}` y la alarma de CloudWatch publica, con el mismo bot, en el chat `OPS_TELEGRAM_CHAT_ID` (§12.4).

#### 21.7 Canales futuros (contrato ya previsto)

| Canal | Notas de diseño |
|---|---|
| **Email** [FUTURO] | Un adaptador más. Entrada vía **SES Receiving** (no disponible en eu-south-2: habría que usar eu-west-1 o un proveedor de *inbound parse*); salida con SES; hilos por `Message-ID` / `In-Reply-To`; sin botones → texto numerado; adjuntos a S3. Ojo: el **canal email** es una dirección propia del bot; **leer el buzón del negocio** sería otra cosa (un conector) |
| Instagram DM / Messenger | Mismo modelo de webhooks y firma de Meta → reutiliza gran parte de WhatsApp; ventana de 24 h; requiere App Review |
| SMS | Conector `notification`, adaptador Twilio; solo como respaldo |
| Voz | Plataforma externa que llama a **endpoints de herramientas** firmados; nuestro backend sigue siendo la fuente de verdad |

---

### 22. Conectores

> **En pocas palabras.** Un conector es la forma estándar de hablar con un sistema externo (Google Calendar, la IA, el email…). Están agrupados por **categoría**: todos los conectores de "reservas" tienen los mismos métodos, sea Google Calendar u otro software. Así un pack pide "dame los huecos del jueves" sin saber qué sistema hay detrás, y **varios packs usan el mismo conector**. Los reintentos, los tiempos máximos y el "fusible" (circuit breaker) se aplican en un solo sitio para todos.

#### 22.1 Piezas

```
Categoría (interfaz con métodos tipados)     core/connectors/interfaces/booking.py
   └─ Adaptador (implementación concreta)     flowpilot/connectors/adapters/google_calendar/
        └─ Manifest (nombre, categoría, esquema de config, credenciales que necesita)
Binding (tenant + categoría → adaptador + config + credencial)   tabla connector_binding
Registro + políticas (reintentos, timeout, circuit breaker, métricas, logs)   flowpilot/connectors/registry.py
```

**Uso desde un pack:**

```python
slots = ctx.connectors.booking.get_slots(day=date(2026,10,9), service=service_spec, rules=rules)
```

`ctx.connectors.booking` es un **proxy** que: resuelve el binding del tenant → descifra la credencial en memoria → llama al adaptador con un timeout → reintenta si el error es transitorio → actualiza el circuit breaker → registra la duración, el resultado y el coste.

#### 22.2 Categorías iniciales y sus métodos

**`booking`** (reservas). Adaptadores: `google_calendar`, `mock`. [FUTURO] `flowww`, `booksy`.

```python
@dataclass(frozen=True)
class ServiceSpec:                     # lo construye el bloque desde ctx.config
    id: str; duration_min: int; client_presence_min: int

@dataclass(frozen=True)
class AvailabilityRules:               # lo construye el bloque desde ctx.config
    weekly_hours: WeeklyHours; slot_step_min: int; min_notice_min: int; timezone: str   # = tenant.timezone

class BookingConnector(Protocol):
    def get_available_days(self, from_date, days, service: ServiceSpec, rules: AvailabilityRules) -> list[date]
    def get_slots(self, day, service: ServiceSpec, rules: AvailabilityRules) -> list[Slot]
    def get_closures(self, from_date, to_date) -> list[Closure]           # lee los eventos [CFG]
    def create_booking(self, req: BookingRequest, idempotency_key: str) -> Booking   # §18.5; SlotTaken
    def cancel_booking(self, booking_id, reason) -> None
    def get_booking(self, booking_id) -> Booking | None
    def check_access(self) -> None                                       # health_check del manifest; lanza error si no hay acceso
    def list_bookings_for_contact(self, contact_id, from_date) -> list[Booking]
    def list_bookings(self, start, end) -> list[Booking]                 # barrido de recordatorios, resumen
    def watch(self, callback_url, token) -> WatchRef                     # WatchRef(gcal_channel_id, gcal_resource_id, expires_at)
    def stop_watch(self, ref: WatchRef) -> None
    def sync_changes(self, sync_token: str | None) -> Changes            # Changes(events, next_sync_token, full)
```

- **Quién calcula los huecos:** `compute_slots` es una función pura de `core/blocks/booking/slots.py`. El adaptador `google_calendar` lee los eventos ocupados y los `[CFG]` del día y la llama con `service` y `rules`. Un adaptador de un software con su propia agenda (tipo Booksy) **ignora `rules`** y pregunta los huecos a su API.
- `create_booking(req)` (con `req.start` y `req.service`) **solo comprueba**, dentro del candado (§18.5), que `[start, start + duration_min)` no se solapa: el resto de reglas ya se aplicaron al ofrecer el hueco.
- [FUTURO] `staff_id` en `get_slots`/`create_booking` cuando haya varios profesionales (un calendario por profesional).

**Google Calendar:**

- Horario semanal, paso y antelación vienen de la config (vía `rules`); cierres y horarios especiales, de los eventos `[CFG]` (§19.7). Sin caché de huecos: siempre se lee de Google.
- Las citas del bot se crean con **id determinista** (`base32hex(sha256(clave))[:32]`, §18.5); los datos de la cita van en la descripción (`Nombre:`, `Telefono:`, `Servicio:`) y en `extendedProperties.private` van `fp_contact_id` y `fp_key`. `list_bookings_for_contact` filtra con `privateExtendedProperty`.
- **Lecturas por ventana** (`get_slots`, `get_closures`, `list_bookings`): `singleEvents=true` + `timeMin`/`timeMax` + paginación con tope.
- **Watch (avisos de cambios)**:
  1. `watch()` llama a `events.watch` con `address = GCAL_WEBHOOK_BASE_URL/webhooks/gcal/<watch_id>` y un `token` aleatorio → se guarda en `calendar_watch` (§23.1, con el token como hash). Justo después se hace una sincronización completa para obtener la línea base.
  2. Los avisos llegan a `POST /webhooks/gcal/<watch_id>`: se verifica `X-Goog-Channel-Token`; el aviso **no trae datos** y **no crea `inbound_event`**: `X-Goog-Resource-State = sync` se ignora; `exists` → `defer(sync_calendar, tenant_id, binding_id, queueing_lock=f"gcal:{binding}")`, que agrupa avisos seguidos en un solo trabajo pendiente.
  3. `sync_calendar` llama a `sync_changes(sync_token)`: **sincronización incremental**, sin `singleEvents`, sin filtro de fechas y sin tope de páginas (`nextSyncToken` solo llega en la última página). Si Google responde **410 Gone** (o no hay `sync_token`), sincronización completa (`Changes.full = true`).
     - Con `full = true` **solo se reconstruye la línea base** y se guarda el `sync_token`: **no se llama al handler** (si no, cada 410 reenviaría confirmaciones de citas antiguas).
     - Con cambios incrementales: por cada evento cambiado se crea `InboundEvent(source=gcal, provider_event_id=<calendar_id>:<event.id>:<event.updated>)` y se encola `process_event` en `inbound` → handler `calendar` con `CalendarChanged` (§17). Al handler solo llegan eventos cuyo `created` es **posterior a `calendar_watch.created_at`** (nada de lo que ya existía al dar de alta el calendario).
  4. **Renovación**: los canales de Google caducan (máx. ~30 días); un trabajo periódico diario (`renew_watches`, con `fp_watches_expiring(48)`) crea un canal nuevo y para el viejo cuando faltan < 48 h, **actualizando la misma fila** de `calendar_watch` (`gcal_channel_id`, `gcal_resource_id`, `expires_at`) y conservando `created_at`.
  5. **Respaldo**: `poll_calendars` (con `fp_bindings_to_poll()`) ejecuta `sync_calendar` cada 60 min aunque no lleguen avisos.
  - [VERIFICAR] que `events.watch` funciona con cuenta de servicio sobre un calendario compartido (§16).

**`llm`** (IA de texto). Adaptadores: `bedrock` (perfil de inferencia EU, rol IAM de la tarea, modelo por configuración), `mock`. Interfaz y reglas en §20.1.

**`email`** (solo salida). Adaptadores: `ses` (staging y prod; SES disponible en eu-south-2 para enviar [VERIFICAR dominio/sandbox]), `smtp` (local, hacia mailpit), `mock`.

```python
class EmailConnector(Protocol):
    def send(self, to: list[str], subject: str, text: str, html: str | None = None,
             reply_to: str | None = None, idempotency_key: str | None = None) -> EmailRef
```

Remitente `SES_FROM` (dominio de la plataforma); `reply_to` configurable por tenant. Lo usa `gym`.

**[FUTURO]** (una línea cada uno, mismo patrón de categoría + adaptador):

- `stt`: voz a texto para audios (`transcribe(media_ref, language)`).
- `sheets`: `append_row`, `read_range` sobre Google Sheets.
- `http`: llamada a una API del cliente declarada en la config (solo HTTPS y dominios permitidos).
- `notification`: SMS (Twilio) como respaldo.

Los **servicios de plataforma** que no varían por tenant (almacenamiento S3, cifrado, reloj) no son conectores: son servicios internos.

#### 22.3 Manifest de un adaptador

```python
# flowpilot/connectors/adapters/google_calendar/manifest.py
manifest = ConnectorManifest(
    name="google_calendar",
    category="booking",
    config_schema=GoogleCalendarConfig,      # calendar_id (la zona horaria es tenant.timezone)
    credentials=CredentialSpec(kind="google_service_account", fields=["json"]),
    factory=GoogleCalendarAdapter,
    policies=Policies(timeout_s=8, retries=3, backoff="exponential", breaker_threshold=5, breaker_cooldown_s=60),
    writes={"create_booking": WriteSpec(on_interrupted="retry"),    # qué hacer con un side_effect cortado (§18.4)
            "cancel_booking": WriteSpec(on_interrupted="retry")},   # cancelar dos veces es inocuo
    health_check="check_access",             # método usado por la comprobación de salud del tenant
)
```

El esquema de config genera el formulario del binding y lo valida. Añadir un adaptador = añadir una carpeta con su manifest: la consola y el registro lo descubren solos.

#### 22.4 Errores y políticas

| Error del adaptador | Significado | Qué hace el registro |
|---|---|---|
| `TransientConnectorError` | Red, 5xx, 429, timeout, lock de reserva no conseguido, `side_effect` en `started` reciente (§18.4) | Reintenta con backoff; cuenta para el circuit breaker (salvo lock y `started`) |
| `PermanentConnectorError` | 4xx, credencial revocada, dato inválido | No reintenta; si es credencial: `NotifyOwner` al dueño y `ops_alert` (§12.4) |
| `ConnectorUnavailable` | Circuit breaker abierto | Falla rápido; el bloque muestra "ahora mismo no puedo consultar la agenda, inténtalo en unos minutos" o deriva a humano |
| Errores de dominio (`SlotTaken`) | Resultado de negocio, no fallo técnico | Se devuelven al bloque, sin reintento ni breaker |

- **Circuit breaker** por (tenant, binding), en memoria de cada proceso (suficiente al principio). El estado se refleja en `integration_health` para la consola.
- **Idempotencia** de las operaciones que escriben (`create_booking`, `cancel_booking`, `email.send`): la clave la genera `ctx.idem` (§18.4).

#### 22.5 Pruebas de conectores

| Nivel | Qué | Dónde |
|---|---|---|
| Unit | Lógica pura (`compute_slots`, parser de descripciones, lectura de `[CFG]`) | CI |
| Contract | El adaptador contra **respuestas reales grabadas** (fixtures JSON de la API, incluidos 409, 410 y paginación de la sincronización) | CI |
| Mock | `mock` de cada categoría, programable por escenario | Tests de packs |
| Smoke | Contra la API real con una cuenta de prueba | Manual en staging, antes de activar un adaptador nuevo |

---

### 23. Datos

> **En pocas palabras.** Toda la información vive en **una base de datos PostgreSQL**. Cada fila que pertenece a un cliente lleva su `tenant_id`, y la propia base de datos impide que un cliente vea datos de otro (Row-Level Security), aunque el código tenga un error.

#### 23.1 Modelo

Todas las tablas marcadas con 🔒 llevan `tenant_id` y RLS. Las claves primarias son **UUIDv7** (ordenables por tiempo; generadas con `uuidv7()` de PostgreSQL 18 o `uuid.uuid7()` de Python 3.14). Las fechas, `timestamptz`.

**Clientes y acceso**

```text
tenant 🔒*            id, slug, name, status[onboarding|active|paused|offboarded], timezone, locale,
                     owner_channel[telegram], llm_monthly_token_quota, created_at
user                 (Django) — tú (operador)
owner_link 🔒         id, tenant_id, telegram_user_id, role[owner|staff], active_tenant, linked_at
owner_notification 🔒 id, tenant_id, owner_link_id, key, kind, text, data jsonb, conversation_id NULL,
                     telegram_message_id, status[pending|sent|failed], created_at   UNIQUE(tenant_id, key)
```
[FUTURO] `tenant_membership` (usuarios Django por tenant, para un portal). `owner_link.active_tenant` es el tenant elegido con `/negocio` (se guarda en todas las filas del mismo usuario de Telegram; se lee con una función `SECURITY DEFINER`, §23.2).
\* `tenant` tiene RLS sobre su propio `id`.

**Packs, configuración y datasets**

```text
pack_version         pack_name, version, checksum, registered_at            (plataforma, sin tenant)
tenant_pack_assignment 🔒  tenant_id, pack_name, config jsonb, config_version,
                          enabled, updated_by, updated_at
config_history 🔒     tenant_id, config_version, config jsonb, changed_by, changed_at
knowledge_doc 🔒      tenant_id, version, content_md, is_active, created_by, created_at
dataset_version 🔒    id, tenant_id, pack, name, version, content jsonb, checksum, is_active,
                     uploaded_by, created_at         UNIQUE(tenant_id, pack, name, version)
                                                     (una sola activa por tenant, pack y name)
form 🔒               id, tenant_id, public_id, pack, status[active|paused], created_at   UNIQUE(public_id)
```

**Canales**

```text
channel 🔒            id, tenant_id, type[whatsapp|telegram|test], status[active|paused|offboarded], public_id, display_name,
                     is_default            (canal principal del tenant por tipo, para SendTemplate)
whatsapp_app 🔒       id, tenant_id, public_id, app_mode[client_app|platform_app], app_id,
                     app_secret_cred, verify_token_hash
whatsapp_connection 🔒  channel_id, tenant_id, whatsapp_app_id, access_token_cred, waba_id,
                       phone_number_id UNIQUE, display_phone_number, quality_rating, messaging_limit_tier
telegram_connection 🔒  channel_id, tenant_id, bot_username, bot_token_cred, secret_token_hash
message_template 🔒   tenant_id, channel_id, name, language, category, status, components jsonb
channel_rate         country, channel_type, category, price_eur, valid_from       (plataforma)
```

**Conectores y credenciales**

```text
credential 🔒         id, tenant_id, kind, ciphertext, encrypted_data_key, kms_key_id, created_at, rotated_at
connector_binding 🔒  id, tenant_id, category, adapter, config jsonb, credential_id, enabled
integration_health 🔒 tenant_id, binding_id, last_ok_at, last_error_at, last_error, breaker_state
calendar_watch 🔒     id, tenant_id, binding_id, gcal_channel_id, gcal_resource_id, token_hash, expires_at,
                     sync_token, last_sync_at, created_at
                     (channel_id y resource_id son los del canal de notificación de Google, no un `channel`
                      nuestro; token se guarda como hash y se compara en tiempo constante)
```

**Conversaciones**

```text
contact 🔒            id, tenant_id, channel_type, external_id, display_name, phone_e164,
                     opt_in_marketing, opt_in_at, opt_out, blocked, last_activity_at, created_at
                                                                          UNIQUE(tenant_id, channel_type, external_id)
conversation 🔒       id, tenant_id, channel_id, contact_id, mode[bot|human], human_until,
                     state jsonb, version, pack_version, last_inbound_at, last_activity_at, opened_at, closed_at
                     UNIQUE(channel_id, contact_id) WHERE closed_at IS NULL   (una sola abierta)
message 🔒            id, tenant_id, conversation_id, direction[in|out], provider_message_id, type,
                     content jsonb, status, pricing_category, billable, cost_eur, created_at
handoff 🔒            id, tenant_id, conversation_id, reason, opened_at, resolved_at, resolved_by
```

**Procesamiento**

```text
inbound_event 🔒      id, tenant_id, channel_id NULL, source[whatsapp|telegram|test|form|gcal|owner_bot|system],
                     provider_event_id, payload jsonb, received_at, status[pending|done|failed|ignored],
                     processed_at, error                UNIQUE(tenant_id, source, provider_event_id)
execution 🔒          id, tenant_id, inbound_event_id, pack, handler, pack_version, status,
                     duration_ms, error_code, llm_tokens_in, llm_tokens_out, cost_eur, trace_id
send_intent 🔒        id, tenant_id, conversation_id, channel_id, seq, payload jsonb,
                     status[pending|sending|sent|failed|unknown], attempts, provider_message_id,
                     last_error, created_at, sent_at
side_effect 🔒        key, tenant_id, kind, status[started|done|failed|unknown], result jsonb, created_at,
                     updated_at                         UNIQUE(tenant_id, key)
scheduled_task 🔒     id, tenant_id, key, task_name, run_at, payload jsonb, conversation_id NULL, max_delay_h,
                     status[scheduled|done|cancelled|failed], job_id, created_at   UNIQUE(tenant_id, key)
```

**Uso y auditoría**

```text
usage_daily 🔒        tenant_id, date, metric[messages_in|messages_out|service_billable|utility|marketing|
                     llm_tokens|executions|errors|emails], count, cost_eur
audit_event          id, tenant_id NULL, actor, action, resource_type, resource_id, at, metadata jsonb
console_action       id, tenant_id NULL, kind, params jsonb, status[pending|running|done|failed], result jsonb,
                     created_by, created_at
tenant_health 🔒      tenant_id, checked_at, status[green|amber|red], details jsonb
owner_link_code 🔒    code_hash, tenant_id, expires_at, used_at
```

**Tablas de plataforma sin RLS** (`audit_event`, `console_action`, y las marcadas como plataforma): solo las lee y escribe `app_operator` (`console_action` la ejecuta el worker con una función `SECURITY DEFINER` que solo lee su fila por id), y `metadata`/`params`/`result` **no contienen datos personales**. `tenant_id` puede ser NULL porque hay acciones que no son de un tenant.

`usage_daily.cost_eur` y `channel_rate` son **métricas de operación** (cuánto cuesta Meta o la IA por tenant), no facturación.

**Tablas de packs**: solo si un pack tiene `storage.py` (§4.2), **siempre con `tenant_id` y RLS**. Ningún pack inicial las tiene: `salon` usa el calendario del cliente y `bus` un dataset.

#### 23.2 Aislamiento entre clientes (RLS)

**Roles de PostgreSQL:**

| Rol | Lo usan | `BYPASSRLS` | Permisos |
|---|---|---|---|
| `app_owner` | Migraciones (tarea `migrate`) | — (dueño de las tablas) | DDL |
| `app_runtime` | `web` (webhooks, bot del dueño, avisos de Calendar, formularios, métrica interna) y `worker` | **No** | SELECT/INSERT/UPDATE/DELETE |
| `app_operator` | Consola, simulador y Django Admin (solo usuarios operador) | **Sí** | SELECT/INSERT/UPDATE/DELETE |

**Política tipo** (se crea en la migración de cada tabla 🔒):

```sql
ALTER TABLE conversation ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON conversation
  USING      (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid)
  WITH CHECK (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid);
```

`NULLIF(…, '')` es necesario: con conexiones persistentes, una vez que una transacción ha usado `SET LOCAL`, `current_setting` devuelve `''` (no NULL) en las siguientes, y `''::uuid` daría error en vez de "0 filas".

**Cómo se fija el tenant:**

- Siempre con **`tenant_tx(tenant_id)`** (§17): abre una transacción corta y ejecuta `SET LOCAL app.tenant_id` (vía `set_config(..., true)`). Cada transacción corta fija el suyo; nunca queda fijado entre transacciones. **Nunca se anida**: `tenant_tx` hace `assert not connection.in_atomic_block`.
- Worker: los trabajos reciben `tenant_id` como argumento y abren tantas `tenant_tx` cortas como necesiten (cargar, guardar `side_effect`, persistir), **nunca una sola transacción alrededor de todo el trabajo** ni alrededor de llamadas externas.
- Entradas HTTP: la vista resuelve el canal (`public_id` de la URL), la app de WhatsApp y el `phone_number_id`, el formulario (`public_id`), el watch (`watch_id`) o el usuario del bot del dueño con funciones SQL `SECURITY DEFINER` que solo devuelven `(tenant_id, tipo, estado)` (y lo mínimo para verificar), y después abre `tenant_tx`.
- **Trabajos que recorren tenants**: obtienen la lista con funciones `SECURITY DEFINER` que solo devuelven ids (`fp_active_tenants(pack)`, `fp_watches_expiring(hours)`, `fp_bindings_to_poll()`) y procesan cada tenant con su propia `tenant_tx` (§18.2).
- Todas las funciones `SECURITY DEFINER` se crean con `SET search_path = pg_catalog, public` (evita que alguien las secuestre con un objeto de otro esquema) y solo con `EXECUTE` para `app_runtime`.
- **Sin tenant fijado no se ve ninguna fila** (`NULLIF(current_setting(..., true), '')` da NULL): si alguien olvida fijarlo, falla de forma segura.
- Django usa dos alias de BD: `default` (`app_runtime`) y `operator` (`app_operator`). Un middleware fija una `ContextVar` `db_alias='operator'` **solo** en peticiones al host `consola.` con un usuario operador autenticado; el *database router* la lee. `DATABASE_URL_OPERATOR` solo existe en la definición de tarea de `web` (el worker no la tiene).
- Las tablas de Procrastinate no tienen `tenant_id` ni RLS: por eso los trabajos solo llevan ids y los candados no llevan datos personales (§18.1).

**Integridad entre tablas:** las tablas padre tienen `UNIQUE(tenant_id, id)` y las hijas usan **claves foráneas compuestas** `(tenant_id, x_id)`. Así es imposible que un mensaje del tenant A apunte a una conversación del tenant B.

**Tests de aislamiento (bloquean el merge):** con tenants A y B, para cada tabla 🔒: A no puede SELECT/UPDATE/DELETE filas de B; no puede insertar con `tenant_id` de B; no puede crear una FK que apunte a B; sin tenant fijado se leen 0 filas; un job de A no ve datos de B. **Un test genérico recorre todas las tablas 🔒 automáticamente**: si alguien añade una tabla con `tenant_id` y olvida la política, el test falla.

#### 23.3 Retención [DECIDIDO inicialmente; se ajusta en el DPA]

| Dato | Retención | Cómo |
|---|---|---|
| `inbound_event.payload` | 30 días | Tarea nocturna (por lotes, §18.2): se vacía el payload y se conservan los metadatos |
| Filas de `inbound_event` | 90 días | Borrado |
| `conversation.state` | Al cerrar la conversación o a los 30 días sin actividad | Se vacía |
| `message.content` | 12 meses | Después, se anonimiza (sin texto, se conservan tipo y fechas) |
| `contact` | 12 meses sin actividad | Borrado (con sus conversaciones, mensajes y derivaciones anonimizados) |
| `handoff` | 12 meses | Se anonimiza (sin motivo en texto libre) |
| `owner_notification` | 6 meses | Borrado |
| Ficheros en S3 (`exports`) | 7 días | Regla de ciclo de vida de S3 |
| `execution`, `send_intent`, `side_effect` | 6 meses | Borrado |
| `dataset_version` no activas | Las 10 últimas por dataset | Borrado de las más antiguas |
| `usage_daily` | Indefinido (agregado, sin datos personales) | — |
| `audit_event` | 3 años | Borrado |
| Tenant dado de baja | Exportación + borrado en ≤ 30 días | Proceso de baja (§13.2) |

#### 23.4 Migraciones de esquema

- Siempre **compatibles con la versión anterior del código** (patrón expand/contract):
  1. *Expand*: añadir la columna o tabla nueva (nullable o con valor por defecto).
  2. Desplegar el código que usa lo nuevo.
  3. *Contract*: en un despliegue posterior, quitar lo viejo.
- Prohibido en un solo paso: renombrar o borrar columnas en uso, `NOT NULL` sin valor por defecto en tablas grandes, cambios de tipo que bloqueen la tabla.
- CI: `makemigrations --check` (no faltan migraciones) + `django-migration-linter` / `squawk` (detecta operaciones peligrosas).
- **Las migraciones que tocan RLS, roles o borran datos requieren revisión humana explícita** (etiqueta `needs-human-db-review` en el PR).

---

### 24. Non-functional requirements

| Clave | Requisito | Implicación arquitectónica |
|---|---|---|
| `nfr.scale` | ~50 negocios, ~75.000 mensajes/mes, picos de 1–2 mensajes/segundo | Un worker con concurrencia 8 da margen de 10 a 50 veces sobre la carga (§14) |
| `nfr.availability` | Pueden caer unos minutos ocasionalmente; no se promete 24/7 | Despliegues sin corte y alarmas (§10.3, §12.4); Multi-AZ solo al tener 3–5 clientes de pago (§11.7) |
| `nfr.latency` | Responder en pocos segundos | El webhook guarda, contesta 200 y encola; el envío va por la cola `outbound` (§2.2, §18.1); los datasets se consultan en memoria (§19.5) |
| `nfr.data_sensitivity` | Datos personales (teléfonos, nombres, emails, citas); sin pagos por ahora; sin datos de salud | Cifrado y RLS obligatorios (§5, §23.2); datos de salud fuera del alcance (§6) |
| `nfr.deployment` | AWS ECS Fargate en eu-south-2 (plan B eu-west-1), CDK en Python, dos cuentas (staging y prod) | Infraestructura como código (§11.5); cuentas separadas (§11.1); región pendiente de verificar (§11.2, §16) |

---

## Testing Strategy

### 25. Calidad y revisión

> **En pocas palabras.** Ningún cambio llega a producción sin pasar por tests automáticos y por tu revisión. Los agentes de IA pueden programar, pero lo que toca datos, seguridad o producción siempre lo aprueba una persona. Cada pack tiene "guiones de conversación" (escenarios) que se prueban automáticamente: si un cambio rompe la reserva de la peluquería, CI lo detecta.

#### 25.1 Tipos de test

| Tipo | Qué prueba | Necesita | Ejemplo |
|---|---|---|---|
| **Unit** | Funciones puras, bloques, plan de envío, matcher, motor de consulta | Nada | `compute_slots`, fusión de mensajes, erratas en localidades |
| **Scenarios** | Conversaciones y handlers completos de un pack, como caja negra | Nada (conectores mock, datasets de ejemplo) | "Reservar corte el jueves a las 17:00" |
| **Datasets** | El validador acepta los ejemplos válidos y rechaza cada caso inválido con su mensaje | Fixtures | Parada desconocida → error con fichero y fila |
| **Integration** | Webhook → cola → worker → envío, con BD real | Postgres | Evento duplicado no se procesa dos veces; dos reservas simultáneas del mismo hueco → una `SlotTaken`; reintento de una reserva ya creada → éxito sin duplicar; sincronización completa tras 410 → no llama al handler; conversación modificada entre tx1 y tx2 → reintento |
| **Isolation** | RLS en todas las tablas 🔒 | Postgres | §23.2 |
| **Contract** | Adaptadores contra respuestas grabadas | Fixtures | Parser de webhooks de Meta, respuestas de Calendar (incl. 410) |
| **Migrations** | No faltan migraciones; no hay operaciones peligrosas | Postgres | — |
| **Smoke** | Tras el despliegue: `/health` + conversación por el canal de pruebas | Entorno desplegado | §10.2 |

[FUTURO] Evals de IA (calidad de respuestas FAQ contra el modelo real) y escenarios por tenant (con snapshots de config exportados desde prod), cuando existan recetas por tenant (§19.6).

#### 25.2 Formato de escenario

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
```

```yaml
# packs/salon/scenarios/reminder_sweep.yaml
name: El barrido envía un recordatorio por cita, una sola vez
fixtures:
  now: "2026-10-08T17:00:00+02:00"
  booking:
    bookings: [{id: ev1, start: "2026-10-09T17:00:00+02:00", phone: "+34600000001", service_id: corte}]
run: {periodic: reminders, times: 2}          # el barrido corre dos veces
expect:
  templates_sent: [{name: reminder, to: "+34600000001"}]   # exactamente una
```

- Cada escenario puede declarar `fixtures` de conectores, datasets de ejemplo (`datasets: {timetable: fixtures/timetable_min.yaml}`) y disparadores distintos del mensaje (`run: {form: …}`, `run: {calendar: …}`, `run: {periodic: …}`).
- Escenarios obligatorios de `salon` además del camino feliz: `slot_taken` (vuelve a ofrecer horas), cierre por `[CFG]`, servicio con `client_presence_min` largo cerca del cierre, cita manual → confirmación.

#### 25.3 Puertas de CI (bloquean el merge)

1. `ruff` (estilo y errores) + `mypy` (tipos).
2. `import-linter` (regla de dependencias §4.2).
3. Tests unit + scenarios + datasets (sin BD; rápidos).
4. Tests integration + isolation + contract (con Postgres 18 de servicio en CI).
5. `makemigrations --check` + linter de migraciones.
6. Validación de packs (`manage.py validate_packs`): recorridos, manifests, esquemas de config con sus configs de ejemplo (y `migrate_config` si hay cambio incompatible, §19.2) y datasets de ejemplo.
7. `pip-audit` (dependencias con vulnerabilidades conocidas).
8. Construcción de la imagen Docker (que compile).

#### 25.4 Flujo de trabajo

Alineado con el framework dev-team (`CLAUDE.md`):

```
IDEA.md → /bootstrap → design.md + spec.md + plan.md + tasks/
  → /orchestrate T-XXX:
       architect (valida la tarea) → planner (plan) → checkpoint humano
       → coder (código + tests, en su rama feature/T-XXX-…) → reviewers (code-quality, security, adversarial…)
  → PR con descripción + checklist → CI en verde
  → REVISIÓN HUMANA (tú) — obligatoria en todos los PR
  → merge a main → /done T-XXX
  → despliegue automático a staging → smoke
  → despliegue a prod (manual, con aprobación)
```

- `main` **no está protegida**: dev-team escribe el estado de las tareas (`tasks/*.md`) directamente en `main`. Todo el **código** entra igualmente por PR con CI en verde y tu revisión.

**Siempre requieren tu aprobación explícita, aunque CI esté en verde:** migraciones que tocan RLS, roles o borran datos; cambios en `credentials/`, KMS, IAM o cifrado; cambios en la verificación de firmas de webhooks; infraestructura (CDK) de prod; cualquier acción sobre datos de prod; activar canales reales de un cliente. Ningún agente despliega a producción ni ejecuta comandos contra la cuenta de prod.

#### 25.5 Revisión de cambios que no son código

- **Config de negocio y knowledge** (Admin o consola): validación por esquema al guardar + historial + auditoría.
- **Datasets**: validador estricto + versiones + activación explícita (§19.5). [FUTURO] vista de revisión y diff para el negocio.
- **Plantillas de WhatsApp**: las revisa Meta; la consola muestra el estado.

---

## Documentation Plan

**Tipo de proyecto:** `mixed` (servicio web en Django, packs de negocio, conectores e infraestructura).

**Documentos que se mantienen:**

- `README.md`: qué es, estado del proyecto y cómo empezar.
- `docs/adr/`: una página por decisión de arquitectura (ADR-001 a ADR-005, enlazadas desde §0):
  - `ADR-001-monolito-modular.md`: monolito modular Django + Procrastinate.
  - `ADR-002-multitenancy-rls.md`: multitenancy con RLS.
  - `ADR-003-packs-packcontext.md`: packs y contrato `PackContext`.
  - `ADR-004-entrada-idempotente.md`: entrada idempotente y "al menos una vez".
  - `ADR-005-aws-ecs-cdk.md`: AWS ECS Fargate + CDK.
- `docs/packs/<pack>.md`: un documento por pack (qué hace, qué configuración pide, qué conectores y datasets necesita).
- `docs/glosario.md`: glosario de términos técnicos en lenguaje llano (fuera de este documento para que no se reparta a los agentes).
- `docs/runbooks/`: procedimientos de operación (§13.4).

**Documento principal de API:** `docs/api.md` (referencia de endpoints: webhooks de canales, avisos de Calendar, formularios públicos, endpoint interno de métricas y `/health`).
