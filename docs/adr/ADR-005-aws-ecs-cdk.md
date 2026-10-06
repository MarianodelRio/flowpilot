# ADR-005 — AWS ECS Fargate + CDK

**Estado:** aceptada · **Relacionado:** `design.md` §0, §10, §11

## Contexto

Hace falta una infraestructura gestionada, barata al principio, sin servidores que mantener, reproducible y con datos en la UE (preferiblemente en España), operable por una persona.

## Decisión

- **AWS**: ECS Fargate (ARM64) para `web` y `worker`, RDS PostgreSQL 18, ALB, S3, KMS, SSM, CloudWatch, EventBridge, SES (envío) y Bedrock (perfil EU).
- Región **eu-south-2 (España)** [VERIFICAR servicios]; plan B **eu-west-1**.
- **Dos cuentas** (staging y prod) bajo AWS Organizations; acceso por IAM Identity Center + MFA; CI por OIDC.
- **Infraestructura como código con AWS CDK en Python**; recursos con datos con protección contra borrado.
- Despliegue progresivo con circuit breaker y rollback automático; staging automático, prod con aprobación manual.
- Sin NAT Gateway al principio (tareas con IP pública y security group cerrado).

## Alternativas consideradas

- **Kubernetes (EKS)**: potencia que no necesitamos y coste de operación alto.
- **VPS / EC2 con Docker**: más barato, pero parches, backups y alta disponibilidad a mano.
- **Terraform**: válido; CDK en Python mantiene un solo lenguaje en el proyecto.
- **PaaS (Render, Fly)**: menos control sobre la región y el cumplimiento.

## Consecuencias

- Coste inicial ~110 $/mes en prod; crece por pasos (Multi-AZ, más tareas, NAT).
- Fargate da como máximo 120 s al parar una tarea: los trabajos largos deben ser reanudables.
- El código no depende de AWS salvo detrás de interfaces (cifrado, almacenamiento, IA, email).

## Cuándo revisarla

- eu-south-2 no ofrece algún servicio necesario → eu-west-1.
- Más de ~8 tareas → subredes privadas + NAT; 20+ clientes → WAF, backups en otra región, Savings Plans.
