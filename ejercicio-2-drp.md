# Ejercicio 2: Plan de Recuperación ante Desastres (DRP)

## Contexto

**LogísticaGlobal** opera una plataforma transaccional en AWS, con una región primaria y una región secundaria en arquitectura **Warm Standby**. Los objetivos son **RPO < 15 minutos** y **RTO < 1 hora**, pero mantener la región secundaria activa genera costos elevados. Además, no hay políticas formales de backup en la región primaria: no existe ciclo de vida, no se trasladan respaldos antiguos a almacenamiento más barato y no se verifica que los respaldos puedan restaurarse.

### Arquitectura actual (Warm Standby)

```
                      Usuarios
                          |
                  Route 53 (DNS con failover)
                  /                       \
     REGIÓN PRIMARIA               REGIÓN SECUNDARIA
     (operación normal)            (Warm Standby)
            |                             |
Application Load Balancer     Application Load Balancer
            |                             |
Auto Scaling Group (EC2)      Instancias EC2 activas
                              con capacidad reducida
            |                             |
    Amazon RDS Multi-AZ          RDS réplica o standby
            |                             |
    Amazon S3 primario  --->     Amazon S3 replicado
```

## Consigna

Optimizar los costos de la estrategia de DR sin afectar el RPO ni el RTO, proponiendo una mejora basada en **Pilot Light**.

---

## 1. Warm Standby → Pilot Light

En Pilot Light solo queda encendido lo que no se puede recrear rápido (los **datos**). El cómputo se apaga y se reconstruye automáticamente durante el failover.

| Componente (región secundaria) | Hoy (Warm Standby) | Propuesta (Pilot Light) | Efecto en RPO/RTO |
|---|---|---|---|
| **Instancias EC2** | Encendidas 24/7 con capacidad reducida | **Apagarlas.** Crear un Auto Scaling Group con capacidad deseada en 0, un Launch Template y **AMIs copiadas** desde la región primaria (copia automática periódica). Al activar el DR, el ASG escala a la capacidad de producción | Suma unos minutos de arranque al RTO, dentro de la hora |
| **Capacidad de cómputo reservada** | Reservada para la secundaria | **Liberarla.** Usar Savings Plans para la base de cómputo y pedir aumento de *service quotas* de EC2 en la región secundaria | Sin impacto, si se prueba el failover |
| **RDS réplica o standby** | Dimensionada para recibir tráfico | **Es el "piloto": se mantiene la replicación continua** (réplica cross-region de RDS o Aurora Global). Usar una **clase de instancia mínima** y, al promover, escalar la instancia y activar Multi-AZ | **El RPO se mantiene.** Agregar una alarma de `ReplicaLag` para detectar atrasos antes de los 15 min |
| **Application Load Balancer** | Activo con targets | **Dejarlo creado y sin targets** (costo fijo bajo) o crearlo por IaC en el failover. Los targets se registran solos cuando el ASG escala | Sin impacto significativo |
| **Amazon S3 replicado** | Replicación general | **Mantener CRR con Replication Time Control** (objetivo de 15 min) solo para datos críticos. Aplicar lifecycle en el bucket destino y excluir logs y temporales | **RPO < 15 min** |
| **Route 53 con failover** | Ya configurado | Mantenerlo. Configurarlo para devolver el registro secundario aunque todavía no tenga instancias sanas, y bajar el TTL | Reduce el tiempo de conmutación |
| **Componentes de red** | Activos | Mantener VPC, subredes, route tables y Security Groups (no tienen costo). Si hubiera NAT Gateways, eliminarlos y crearlos por IaC en el failover | Suma pocos minutos |

### Secuencia de failover (RTO < 1 hora)

1. **Route 53** detecta la caída de la región primaria y se activa el runbook (alarma → Systems Manager Automation o Step Functions).
2. Se **promueve la réplica de RDS** y se escala su instancia.
3. El **Auto Scaling Group** pasa de 0 al tamaño de producción usando las AMIs ya copiadas.
4. Los targets se registran en el ALB y pasan los health checks.
5. **Route 53** envía el tráfico a la región secundaria.
6. Se validan los servicios y se comunica la recuperación.

> Todo debe estar definido como **infraestructura como código** (CloudFormation o Terraform) y probado con **game days** periódicos, porque es lo que respalda el RTO < 1 hora. Los tiempos de cada paso son estimaciones y deben medirse en pruebas reales.

### Ahorro esperado

Desaparece el costo del cómputo permanente (EC2 encendidas y capacidad reservada) y de la base de datos sobredimensionada. Se sigue pagando la réplica mínima, el almacenamiento replicado, las AMIs y el ALB.

---

## 2. Optimización de backups en la cuenta primaria

| Eje | Medida |
|---|---|
| **Gobierno** | Centralizar todo en **AWS Backup** con *backup plans* y asignación por **tags** (`criticidad=alta/media/baja`), en lugar de snapshots manuales sueltos. |
| **Frecuencia** | Según criticidad. RDS: **PITR continuo** más snapshot diario. EBS y recursos críticos: diario. Baja criticidad: semanal. El RPO del DR lo cubre la replicación, así que los backups no necesitan ser cada 15 minutos: sirven para recuperar ante borrado, corrupción o ransomware. |
| **Retención escalonada** | Ejemplo: diarios 7-35 días, semanales 4-12 semanas, mensuales 12 meses, anuales solo si hay requisito legal. Todo backup con fecha de expiración. |
| **Almacenamiento y ciclo de vida** | En AWS Backup, pasar a **cold storage** los respaldos antiguos donde el recurso lo soporte (mínimo 90 días en frío). En respaldos en S3: Standard → Standard-IA → Glacier → Deep Archive, con expiración final. |
| **Copias entre regiones** | Copiar a la secundaria **solo los backups críticos**. |
| **Limpieza** | Eliminar snapshots huérfanos y AMIs obsoletas, y monitorear costos con Cost Explorer por tags. |
| **Verificación** | **AWS Backup restore testing** automático y periódico, más una revisión trimestral documentada. Un backup sin probar no garantiza recuperación. |
| **Seguridad** | Cifrado con KMS y **Backup Vault Lock** (inmutabilidad). |

---

## Conclusión

Pilot Light conserva el **RPO** (replicación continua de RDS y S3) y el **RTO** (automatización probada) y elimina el costo del cómputo permanente. Una política formal de backups, con retención escalonada, ciclo de vida y pruebas de restauración, reduce el costo de almacenamiento y asegura que los respaldos sirvan cuando se necesiten.
