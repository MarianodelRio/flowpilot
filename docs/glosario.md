# Glosario

> Términos técnicos de `design.md` explicados en lenguaje llano.

| Término | Qué es (en llano) |
|---|---|
| **Advisory lock** | Candado de PostgreSQL que la aplicación toma por un nombre (p. ej. "calendario X, día Y") para que dos procesos no hagan lo mismo a la vez |
| **ALB** (Application Load Balancer) | La "puerta de entrada" de AWS: recibe las peticiones HTTPS y las reparte entre las copias de la web |
| **API** | La forma en que un programa le pide cosas a otro (p. ej. "envía este mensaje" a Meta) |
| **Autoescalado** | AWS añade o quita copias del programa según la carga |
| **Backoff exponencial** | Reintentar esperando cada vez más (2 s, 4 s, 8 s…) para no saturar |
| **Bedrock** | Servicio de AWS que da acceso a modelos de IA de varios proveedores; con un perfil EU, el tráfico se queda en la UE |
| **Canal de pruebas** | Canal interno (`test`) para conversar con el bot desde la consola, sin WhatsApp ni Telegram |
| **CDK** | Herramienta para describir la infraestructura de AWS como código Python |
| **CI/CD** | Integración continua: tests automáticos en cada cambio. Despliegue continuo: publicar automáticamente |
| **Circuit breaker** (fusible) | Si un servicio externo falla mucho, se deja de llamarlo un rato para no empeorar las cosas |
| **CloudWatch** | El sistema de logs, métricas y alarmas de AWS |
| **Contenedor / imagen** | La imagen es el programa empaquetado con todo lo que necesita; un contenedor es esa imagen ejecutándose |
| **Dataset** | Datos estructurados de un negocio (p. ej. horarios de autobús) que la plataforma valida, versiona y consulta en memoria |
| **DSL** | Lenguaje propio para describir algo (p. ej. flujos en YAML con condiciones); lo evitamos: los recorridos se escriben en Python |
| **ECR** | Almacén de imágenes de contenedor de AWS |
| **ECS / Fargate** | El servicio de AWS que ejecuta contenedores; con Fargate no hay servidores que gestionar |
| **EMF** (Embedded Metric Format) | Formato de log JSON que CloudWatch convierte automáticamente en métricas |
| **Expand/contract** | Cambiar la BD en dos pasos para que la versión vieja y la nueva del código funcionen a la vez |
| **HMAC / firma** | Un código que demuestra que un mensaje viene de quien dice (Meta) y no ha sido modificado |
| **HTMX** | Librería que permite páginas web interactivas sin escribir JavaScript |
| **Idempotente** | Que hacerlo dos veces tiene el mismo efecto que hacerlo una |
| **KMS** | Servicio de AWS que guarda claves de cifrado; las claves nunca salen de él |
| **LLM** | Modelo de lenguaje (la IA de texto); aquí, servido por Amazon Bedrock |
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
| **SES** | Servicio de envío de email de AWS |
| **SSM Parameter Store** | Almacén de configuración y secretos de AWS |
| **Tenant** | Un cliente (negocio) de la plataforma |
| **UUIDv7** | Identificador único que además se ordena por fecha de creación |
| **Watch / syncToken** | *Watch*: suscripción a los cambios de un Google Calendar (Google avisa a nuestra URL). *syncToken*: marca que permite pedir a Google "solo lo que ha cambiado desde la última vez" |
| **Webhook** | Aviso que un servicio externo (Meta, Telegram, Google) envía a nuestra URL cuando pasa algo |
| **Worker** | El proceso que hace el trabajo de fondo (lo que no es atender una petición web) |
