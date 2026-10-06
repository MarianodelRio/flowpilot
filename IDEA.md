# FlowPilot — Idea

## ¿Qué problema resuelve?

Los pequeños negocios (peluquerías, autobuses, gimnasios) pierden horas cada semana contestando WhatsApps, gestionando reservas a mano y copiando datos entre herramientas que no se hablan entre sí.

FlowPilot automatiza esas tareas repetitivas: atiende a los clientes por WhatsApp, gestiona reservas y avisa al dueño cuando hace falta.

## ¿Quién lo usa?

- **Dueños de pequeños negocios** (1–15 empleados) y **su clientela**, que escribe por WhatsApp.
- **El desarrollador** (Mariano), como operador de la plataforma: da de alta negocios, configura packs y vigila que todo funcione.

## ¿Cómo funciona a alto nivel?

1. El cliente escribe por WhatsApp (por ejemplo, "quiero cita el sábado por la mañana").
2. FlowPilot entiende la petición, consulta su agenda o sus datos (Google Calendar, etc.), responde al cliente y deja registro de lo ocurrido.
3. El dueño recibe avisos por Telegram (nueva cita, cancelación, algo que requiere su atención).

## ¿Hay algo técnico que ya quieres?

- **Lenguaje y framework:** Python + Django.
- **Base de datos:** PostgreSQL.
- **Colas de trabajo:** en Postgres, con Procrastinate.
- **Infraestructura:** AWS — ECS Fargate, RDS, región eu-south-2 (Milán), con CDK en Python.
- **Aislamiento:** los datos de cada negocio están separados del resto (tenant) y la base de datos lo refuerza con RLS.

## ¿Qué NO es parte de esto?

- **No es un chatbot de IA de propósito general.** La IA solo interpreta peticiones dentro de los flujos de cada negocio.
- **No es un software de reservas que compita con Booksy.** FlowPilot se conecta a la agenda que el negocio ya usa.
- **No es un CRM.**
- **No incluye facturación propia.**
- **No es una app móvil.** Los dueños usan Telegram y los clientes usan WhatsApp.
- **No ofrece disponibilidad 24/7.** Puede haber caídas de unos minutos de vez en cuando.
- **No usa SMS como canal principal.**

## Imprescindible el primer día

- Reservar y cancelar citas en una peluquería, con Google Calendar.
- Recordatorios de cita.
- Aviso al dueño por Telegram.
- Aislamiento total entre negocios: los datos de un negocio nunca son visibles para otro.
