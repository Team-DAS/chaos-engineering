# Chaos Engineering - UdeAJobs Cell-Based Architecture

## Objetivo

Validar las propiedades de aislamiento y resiliencia de la arquitectura basada en células mediante experimentos controlados de inyección de fallos con Chaos Mesh.

## Propiedades Validadas

- **Aislamiento de Fallos**: Blast radius = 0% (fallos no se propagan entre células)
- **Recuperación Automática**: Kubernetes detecta y recupera pods mediante health checks
- **Aislamiento de Recursos**: Resource Limits contienen CPU/memoria dentro de namespaces
- **Independencia Operacional**: Células no afectadas mantienen operación normal

## Stack Tecnológico

- Chaos Mesh v2.6+
- Kubernetes v1.28 en Google Kubernetes Engine
- Prometheus + Grafana para observabilidad
- 4 células: identity-cell, profile-cell, projects-cell, lifecycle-cell

## Experimentos

### 1. Pod Failure - Identity Cell
**Archivo**: `experiments/pod-failure-identity.yaml`  
**Tipo**: PodChaos (pod-kill)  
**Objetivo**: Validar recuperación automática y aislamiento de fallos

```yaml
action: pod-kill
mode: one
selector:
  namespaces: [identity-cell]
duration: "30s"
```

### 2. Pod Failure - Projects Cell
**Archivo**: `experiments/pod-failure-projects.yaml`  
**Tipo**: PodChaos (pod-kill)  
**Objetivo**: Confirmar consistencia de aislamiento en célula diferente

```yaml
action: pod-kill
mode: one
selector:
  namespaces: [projects-cell]
duration: "30s"
```

### 3. CPU Stress - Profile Cell
**Archivo**: `experiments/cpu-stress-profile.yaml`  
**Tipo**: StressChaos  
**Objetivo**: Validar contención de CPU mediante Resource Limits

```yaml
stressors:
  cpu:
    workers: 2
    load: 100
duration: "60s"
```

### 4. Memory Stress - Lifecycle Cell
**Archivo**: `experiments/memory-stress-lifecycle.yaml`  
**Tipo**: StressChaos  
**Objetivo**: Validar aislamiento de memoria entre células

```yaml
stressors:
  memory:
    workers: 1
    size: "512MB"
duration: "60s"
```

## Instalación de Chaos Mesh

```bash
# Agregar repo y crear namespace
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
kubectl create namespace chaos-mesh

# Instalar Chaos Mesh
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace=chaos-mesh \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --set dashboard.create=true

# Verificar instalación
kubectl get pods -n chaos-mesh
```

## Ejecución de Experimentos

```bash
# Aplicar experimento
kubectl apply -f experiments/[experimento].yaml

# Monitorear
watch -n 1 'kubectl get pods -n [namespace]'
kubectl get events -n [namespace] --sort-by='.lastTimestamp'

# Limpiar
kubectl delete -f experiments/[experimento].yaml
```

## Verificación de Blast Radius

```bash
# Verificar que otras células NO tienen eventos de fallo
kubectl get events -n identity-cell --sort-by='.lastTimestamp' | tail -10
kubectl get events -n profile-cell --sort-by='.lastTimestamp' | tail -10
kubectl get events -n projects-cell --sort-by='.lastTimestamp' | tail -10
kubectl get events -n lifecycle-cell --sort-by='.lastTimestamp' | tail -10

# Verificar restart counts
kubectl get pods -A | grep -E 'identity-cell|profile-cell|projects-cell|lifecycle-cell'
```

## Dashboard de Grafana

Queries recomendadas para monitoreo:

**Running Pods**:
```promql
count(kube_pod_status_phase{phase="Running", namespace=~"identity-cell|profile-cell|projects-cell|lifecycle-cell"}) by (namespace)
```

**CPU Usage**:
```promql
sum(rate(container_cpu_usage_seconds_total{namespace=~"identity-cell|profile-cell|projects-cell|lifecycle-cell", container!=""}[5m])) by (namespace)
```

**Memory Usage**:
```promql
sum(container_memory_working_set_bytes{namespace=~"identity-cell|profile-cell|projects-cell|lifecycle-cell", container!=""}) by (namespace)
```

**Pod Restarts**:
```promql
sum(kube_pod_container_status_restarts_total{namespace=~"identity-cell|profile-cell|projects-cell|lifecycle-cell"}) by (namespace)
```

## Estructura del Repositorio

```
chaos-engineering/
├── README.md
├── experiments/
│   ├── pod-failure-identity.yaml
│   ├── pod-failure-projects.yaml
│   ├── cpu-stress-profile.yaml
│   └── memory-stress-lifecycle.yaml
```

## Nota sobre Observabilidad

Si la recuperación ocurre en <10 segundos, puede no ser visible en Grafana debido al intervalo de scraping de Prometheus (15-30s). **Esto es esperado y normal**. Evidencias válidas incluyen:
- Eventos de Kubernetes
- Restart counters
- Logs de Chaos Mesh

## Referencias

- **Proyecto UdeAJobs**: https://github.com/Team-DAS
- **Documentación**: https://team-das.github.io/Documentation/
- **Chaos Mesh**: https://chaos-mesh.org/docs/
- **Principles of Chaos Engineering**: https://principlesofchaos.org/

## Equipo

**Universidad de Antioquia - Ingeniería de Sistemas**

- Argenis Medina Morales
- David Gomez Agudelo
- Santiago Trespalacios Bolivar

**Tutor**: Diego Botia Valderrama

Proyecto Integrador I - 2025-2
