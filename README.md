# Ejercicio 1: Diagnóstico de incidentes mediante responsabilidad compartida

## Contexto

La startup **CloudEdu** opera una plataforma web de servicios digitales en AWS, usando infraestructura y servicios administrados para ejecutar la aplicación, almacenar información y procesar las operaciones de sus usuarios. Durante la última semana se registraron tres incidentes independientes.

| # | Incidente |
|---|-----------|
| 1 | Un integrante del equipo eliminó por error la base de datos productiva (Amazon RDS, servicio administrado tipo PaaS). No había backups automáticos ni política de retención y recuperación. |
| 2 | Una falla física de gran escala afectó el centro de datos del proveedor durante varias horas. La empresa no tenía estrategia de recuperación en otra región. |
| 3 | Un atacante accedió a una máquina virtual (Amazon EC2, IaaS) que tenía una contraseña predeterminada y el puerto SSH abierto a todo Internet. |

## Consigna

Para cada incidente: determinar la responsabilidad principal (proveedor o cliente), identificar la capa de servicio, fundamentar técnicamente según el modelo de responsabilidad compartida e indicar medidas de mitigación.

---

## Resumen

| Incidente | Responsabilidad principal | Capa de servicio |
|---|---|---|
| **1. Eliminación de la BD sin backups** | **Cliente (CloudEdu)** | **PaaS** (Amazon RDS) |
| **2. Falla física del centro de datos** | **Proveedor (AWS)** por la falla; **Cliente** por la falta de resiliencia | **Infraestructura física/global** (base de IaaS, común a todas las capas) |
| **3. Acceso a EC2 por contraseña por defecto y SSH abierto** | **Cliente (CloudEdu)** | **IaaS** (Amazon EC2) |

---

## Incidente 1: Eliminación de la base de datos productiva

**Responsabilidad principal:** Cliente (CloudEdu)
**Capa de servicio:** PaaS (Amazon RDS)

### Fundamentación técnica

En PaaS, AWS administra el hardware, el sistema operativo, el motor de base de datos y el parcheo, y ofrece la *capacidad* de realizar backups. Sin embargo, la **configuración del servicio, la gestión de los datos y de los accesos (IAM)** son responsabilidad del cliente.

AWS no eliminó nada: lo hizo un integrante del equipo. La ausencia de backups y de una política de retención y recuperación es una decisión de configuración y gobierno que le corresponde al cliente. Es responsabilidad **"en" la nube**, no **"de" la nube**.

### Medidas preventivas

- Habilitar **backups automáticos** y snapshots manuales con política de retención definida (RPO/RTO).
- Activar **deletion protection** en la instancia.
- Aplicar **mínimo privilegio** con IAM y MFA, restringiendo `rds:DeleteDBInstance`.
- Separar roles y entornos (producción vs. desarrollo).
- Exigir **snapshot final** al eliminar y **probar restauraciones** periódicamente.
- Auditar acciones con **CloudTrail** y configurar alertas.

---

## Incidente 2: Falla física del centro de datos

**Responsabilidad principal:** Proveedor (AWS) por la falla física; Cliente por la falta de resiliencia
**Capa de servicio:** Infraestructura física/global (base de IaaS, común a todas las capas)

### Fundamentación técnica

AWS es responsable de la **seguridad y disponibilidad de la infraestructura física**: centros de datos, energía, refrigeración, red y hardware. Por eso la falla en sí es suya.

Sin embargo, AWS ofrece múltiples Zonas de Disponibilidad (AZ) y regiones, y el **diseño de una arquitectura resiliente es responsabilidad del cliente**. CloudEdu dependía de una única ubicación y no tenía estrategia de recuperación ante desastres (DR), por lo que el *impacto* es consecuencia de su propio diseño.

### Medidas preventivas

- Arquitectura **Multi-AZ** (balanceador de carga, Auto Scaling, RDS Multi-AZ).
- Estrategia de **DR multi-región** (backup & restore, pilot light, warm standby o active-active, según RTO/RPO requerido).
- **Replicación de datos entre regiones** (S3 CRR, réplicas de lectura, Aurora Global).
- **Infraestructura como código** para reconstruir rápidamente el entorno.
- **Plan de continuidad del negocio** con pruebas periódicas (game days).

---

## Incidente 3: Acceso a EC2 por contraseña predeterminada y SSH abierto

**Responsabilidad principal:** Cliente (CloudEdu)
**Capa de servicio:** IaaS (Amazon EC2)

### Fundamentación técnica

En IaaS, AWS gestiona el hardware, la virtualización y la red base. El cliente es responsable del **sistema operativo, parches, configuración, credenciales, aplicaciones, firewall (Security Groups) y control de acceso**.

La contraseña por defecto y el puerto 22 abierto a `0.0.0.0/0` son fallas de configuración del cliente. No hay ninguna vulnerabilidad en la infraestructura de AWS.

### Medidas preventivas

- Eliminar contraseñas por defecto; usar **autenticación por claves SSH** y deshabilitar el login por contraseña.
- Restringir el Security Group del puerto 22 a **IPs específicas** o usar un **bastion host**.
- Preferir **AWS Systems Manager Session Manager** (sin puertos abiertos).
- **Parches y hardening** del sistema operativo; usar **IMDSv2**.
- Monitoreo con **GuardDuty, CloudTrail y VPC Flow Logs**.
- Revisión periódica con **AWS Config / Security Hub** y principio de mínimo privilegio.

---

## Conclusión

Los incidentes 1 y 3 son fallas de **configuración y gobierno del cliente**, que le corresponden aun cuando el servicio sea administrado (PaaS) o de infraestructura (IaaS).

El incidente 2 muestra la **responsabilidad compartida real**: AWS responde por la seguridad *de* la nube (infraestructura física), mientras que CloudEdu responde por la seguridad y la resiliencia *en* la nube (arquitectura, DR y datos).
