# Ejercicio 3: Diagnóstico de resiliencia, monitoreo y pipelines CI/CD inmutables

## Contexto

En la fintech **PagoDigital**, una falla de conectividad en el microservicio secundario de *Notificaciones por Correo* provocó que el microservicio principal de *Procesamiento de Pagos* se quedara esperando respuestas indefinidamente. Al agotarse los hilos de ejecución y la memoria, se produjo una caída en cascada (efecto dominó). Durante la emergencia, el operador de guardia ingresó por SSH al servidor y modificó el código y las dependencias directamente dentro de los contenedores Docker en ejecución.

**Causa raíz:** Procesamiento de Pagos llamaba a Notificaciones sin timeout ni circuit breaker. Cada llamada colgada retenía un hilo hasta agotar el pool y la memoria, y la falla se propagó en cascada. Después, el parche manual dentro del contenedor agregó un segundo problema: la modificación en caliente no es reproducible ni auditable.

### Pipeline objetivo

```
[ Developer Push ] --> [ GitHub Actions ] --> [ Docker Build & Tag ] --> [ Container Registry ] --> [ K8s Deploy ]
```

---

## Eje 1: Monitoreo proactivo y alarmas

### Métrica principal: saturación del pool de hilos del servicio de pagos

Es la métrica que mejor anticipa este incidente, porque los hilos se fueron ocupando de a poco antes de la caída.

```promql
# Ejemplo para una app Java/Spring (Tomcat); adaptar el nombre según el runtime
tomcat_threads_busy_threads{app="procesamiento-pagos"}
  / tomcat_threads_config_max_threads{app="procesamiento-pagos"}
```

| Severidad | Umbral | Duración | Acción |
|---|---|---|---|
| **Warning** | > **70 %** de hilos ocupados | 5 min | Aviso por Slack/email al equipo |
| **Critical** | > **85 %** de hilos ocupados | 2 min | Page al on-call |

### Métricas complementarias

| Métrica | Fuente | Umbral sugerido |
|---|---|---|
| **Latencia p95 de las llamadas a Notificaciones** | Prometheus (métricas del cliente HTTP) | > 2 s durante 3 min |
| **Memoria del contenedor / límite** | cAdvisor: `container_memory_working_set_bytes` | > 85 % del límite durante 5 min |
| **Tasa de 5xx y latencia p95 en el Ingress** | Logs/métricas de NGINX Ingress Controller | 5xx > 5 % durante 2 min; p95 > 3 s |

### Regla de alerta (Prometheus)

```yaml
groups:
  - name: pagos-saturacion
    rules:
      - alert: PagosHilosSaturados
        expr: |
          tomcat_threads_busy_threads{app="procesamiento-pagos"}
            / tomcat_threads_config_max_threads{app="procesamiento-pagos"} > 0.85
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Pool de hilos de Pagos por encima del 85 %"
```

Con el warning al 70 %, el equipo habría tenido tiempo de actuar antes de que se agotaran los hilos. Un dashboard en Grafana con estas métricas, más la latencia hacia Notificaciones, habría mostrado dónde se estaba originando el problema.

### Medidas de resiliencia (para evitar la cascada)

- **Timeouts** de conexión y lectura en toda llamada entre servicios.
- **Circuit breaker** (por ejemplo Resilience4j): si Notificaciones falla, se abre el circuito y Pagos sigue operando.
- **Bulkhead:** pool de hilos separado para las llamadas a Notificaciones.
- **Desacoplamiento asíncrono:** las notificaciones salen por una cola (SQS/RabbitMQ), de modo que un correo caído no afecta un pago.

---

## Eje 2: Despliegues inmutables y CI/CD

### ¿Qué es la inmutabilidad?

Un artefacto desplegado (la imagen de contenedor) **nunca se modifica después de construirse**. Para cambiar algo se construye una **imagen nueva con un tag nuevo** y se reemplazan los contenedores por los de la nueva versión. Nadie edita contenedores en ejecución.

**Por qué el parche por SSH fue un error:**

- Genera **configuration drift**: ese contenedor ya no coincide con la imagen ni con el repositorio.
- Se **pierde al reiniciar** el pod, y en cada réplica el código queda distinto.
- No hay revisión, tests ni **auditoría** del cambio.
- Expone acceso SSH innecesario en producción.

**Beneficios del enfoque inmutable:** reproducibilidad, trazabilidad (cada versión se asocia a un commit) y **rollback inmediato** volviendo al tag anterior.

### Docker: imagen inmutable sin `latest`

```bash
# Tag = versión + SHA corto del commit (trazable a un único estado del código)
IMAGE=ghcr.io/pagodigital/procesamiento-pagos
TAG=1.4.2-$(git rev-parse --short HEAD)    # ej: 1.4.2-a1b2c3d

docker build -t $IMAGE:$TAG .
docker push $IMAGE:$TAG
```

`latest` es un tag mutable: puede apuntar a imágenes distintas en momentos distintos, por lo que no se puede saber qué versión corre ni volver atrás con certeza. Para máxima inmutabilidad se puede fijar además el **digest** (`image@sha256:...`).

### GitHub Actions: `.github/workflows/deploy.yml`

Los tres steps mínimos son **checkout, build & push y deploy**. El login al registry y la configuración del acceso al cluster son pasos de soporte necesarios para que funcionen.

```yaml
name: CI/CD Pagos
on:
  push:
    branches: [main]

env:
  IMAGE: ghcr.io/pagodigital/procesamiento-pagos

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      # 1) Checkout del código
      - name: Checkout
        uses: actions/checkout@v4

      # Soporte: login al Container Registry
      - name: Login a GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # 2) Build & push con tag inmutable (SHA del commit)
      - name: Build y push de la imagen
        run: |
          docker build -t $IMAGE:${{ github.sha }} .
          docker push $IMAGE:${{ github.sha }}

      # Soporte: acceso al cluster
      - name: Configurar kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config

      # 3) Deploy a Kubernetes con la nueva imagen
      - name: Deploy a Kubernetes
        run: |
          kubectl set image deployment/procesamiento-pagos \
            pagos=$IMAGE:${{ github.sha }} -n produccion
          kubectl rollout status deployment/procesamiento-pagos -n produccion
```

Las credenciales van como **GitHub Secrets**, nunca en el repositorio.

### Kubernetes: estrategia RollingUpdate

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: procesamiento-pagos
  namespace: produccion
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # permite 1 pod extra durante la actualización
      maxUnavailable: 0    # nunca baja de la capacidad actual: sin pérdida de disponibilidad
  selector:
    matchLabels:
      app: procesamiento-pagos
  template:
    metadata:
      labels:
        app: procesamiento-pagos
    spec:
      containers:
        - name: pagos
          image: ghcr.io/pagodigital/procesamiento-pagos:a1b2c3d
          readinessProbe:          # el pod recibe tráfico solo cuando está listo
            httpGet: { path: /health/ready, port: 8080 }
          livenessProbe:
            httpGet: { path: /health/live, port: 8080 }
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "500m", memory: "1Gi" }
```

**Comandos equivalentes por terminal:**

```bash
# Actualizar sin perder disponibilidad
kubectl set image deployment/procesamiento-pagos \
  pagos=ghcr.io/pagodigital/procesamiento-pagos:a1b2c3d -n produccion

# Verificar el progreso
kubectl rollout status deployment/procesamiento-pagos -n produccion

# Rollback si algo falla (vuelve a la imagen anterior)
kubectl rollout undo deployment/procesamiento-pagos -n produccion
```

Con `maxUnavailable: 0` y la **readinessProbe**, Kubernetes levanta un pod nuevo, espera a que esté listo y recién entonces baja uno viejo, así que los pagos siguen procesándose durante todo el despliegue.

---

## Conclusión

El incidente combinó dos fallas: **falta de resiliencia** (sin timeouts ni circuit breaker) y **falta de visibilidad** (sin alertas de saturación), y la respuesta manual por SSH agregó drift. Con alertas sobre el pool de hilos al 70 % y 85 %, el equipo habría detectado el problema a tiempo. Con un pipeline inmutable, el arreglo se habría entregado como una nueva imagen etiquetada con el commit, desplegada con RollingUpdate y con rollback disponible en un comando.
