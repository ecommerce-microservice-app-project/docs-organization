# Guía de Monitoreo para GKE con Prometheus y Grafana

## Información del Proyecto

**Infraestructura:**
- Proyecto GCP: `tfg-prod-478914`
- Cluster GKE: `gke-prod-cluster` (zona: `us-central1-a`)
- VM Monitoreo: `prod-monitoring-vm`
- IP Externa: `34.41.209.157`

**Versiones:**
- Prometheus: v3.0.1
- Grafana: Latest (apt)
- kube-state-metrics: v2.13.0
- node-exporter: v1.8.2

---

## Arquitectura del Sistema

```
┌───────────────────────────────────────────────────────┐
│                  GKE Cluster (3 nodos)                 │
│  • kube-state-metrics (estado del cluster)            │
│  • node-exporter (métricas del SO)                    │
│  • cAdvisor (recursos de contenedores)               │
└─────────────────────┬─────────────────────────────────┘
                      │ kubectl proxy (puerto 8001)
┌─────────────────────┴─────────────────────────────────┐
│               VM de Monitoreo (10.0.0.2)               │
│  • kubectl proxy → localhost:8001                      │
│  • Prometheus → localhost:9090 (10 targets)           │
│  • Grafana → 0.0.0.0:3000                             │
└────────────────────────────────────────────────────────┘
                      │ HTTP
                      ▼
              [Usuario - Navegador]
```

**Justificación:** Prometheus y Grafana corren en VM externa para no sobrecargar el cluster. kubectl proxy permite acceso seguro a métricas sin exponer servicios públicamente.

---

## Setup Inicial

### 1. Configurar Acceso a la VM

```bash
# Crear regla de firewall para SSH
gcloud compute firewall-rules create allow-ssh \
  --project tfg-prod-478914 \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=0.0.0.0/0

# Conectarse a la VM
gcloud compute ssh prod-monitoring-vm \
  --zone us-central1-a \
  --project tfg-prod-478914
```

### 2. Instalar Herramientas Base en la VM

```bash
# Actualizar sistema
sudo apt-get update && sudo apt-get upgrade -y

# Instalar dependencias
sudo apt-get install -y wget curl git apt-transport-https software-properties-common
```

---

## Instalación de Prometheus

### 1. Descargar e Instalar

```bash
# Crear directorios
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# Descargar y extraer Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.0.1/prometheus-3.0.1.linux-amd64.tar.gz
tar -xvf prometheus-3.0.1.linux-amd64.tar.gz
cd prometheus-3.0.1.linux-amd64

# Copiar binarios y archivos
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus

# Verificar instalación
prometheus --version
```

### 2. Crear Usuario y Servicio systemd

```bash
# Crear usuario
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Crear servicio
sudo tee /etc/systemd/system/prometheus.service > /dev/null << 'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
EOF

# Habilitar e iniciar
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

### 3. Crear Regla de Firewall

```bash
gcloud compute firewall-rules create allow-prometheus \
  --project tfg-prod-478914 \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:9090 \
  --source-ranges=0.0.0.0/0
```

---

## Configuración de kubectl en la VM

### 1. Instalar Google Cloud SDK y kubectl

```bash
# Agregar repositorio
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

# Importar clave
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg

# Instalar
sudo apt-get update
sudo apt-get install -y google-cloud-sdk kubectl google-cloud-sdk-gke-gcloud-auth-plugin
```

### 2. Configurar Acceso al Cluster

```bash
# Inicializar gcloud (seguir instrucciones en pantalla)
gcloud init

# Obtener credenciales del cluster
gcloud container clusters get-credentials gke-prod-cluster \
  --location us-central1-a \
  --project tfg-prod-478914

# Verificar conectividad
kubectl get nodes
```

### 3. Configurar kubectl proxy como Servicio

```bash
# Crear servicio systemd
sudo tee /etc/systemd/system/kubectl-proxy.service > /dev/null << 'EOF'
[Unit]
Description=kubectl proxy for Prometheus
After=network.target

[Service]
Type=simple
User=<tu-usuario>
ExecStart=/usr/bin/kubectl proxy --address=127.0.0.1 --port=8001 --accept-hosts='^localhost$,^127\.0\.0\.1$'
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Habilitar e iniciar
sudo systemctl daemon-reload
sudo systemctl enable kubectl-proxy
sudo systemctl start kubectl-proxy
```

**Nota:** Reemplaza `<tu-usuario>` con tu usuario en la VM.

---

## Despliegue de Componentes en el Cluster

### 1. Desplegar kube-state-metrics

**Desde tu máquina local:**

```bash
# Configurar kubectl localmente
gcloud container clusters get-credentials gke-prod-cluster \
  --location us-central1-a \
  --project tfg-prod-478914

# Aplicar manifiestos
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kube-state-metrics/main/examples/standard/cluster-role-binding.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kube-state-metrics/main/examples/standard/cluster-role.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kube-state-metrics/main/examples/standard/deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kube-state-metrics/main/examples/standard/service-account.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kube-state-metrics/main/examples/standard/service.yaml

# Si los pods quedan en Pending, aplicar tolerations
kubectl patch deployment kube-state-metrics -n kube-system --patch '
spec:
  template:
    spec:
      tolerations:
      - key: "components.gke.io/gke-managed-components"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
'

# Verificar
kubectl get pods -n kube-system | grep kube-state-metrics
```

### 2. Desplegar node-exporter

```bash
# Crear manifiesto
cat > node-exporter-daemonset.yaml << 'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - key: "components.gke.io/gke-managed-components"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.8.2
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
        - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
        ports:
        - containerPort: 9100
          name: metrics
        resources:
          limits:
            cpu: 150m
            memory: 120Mi
          requests:
            cpu: 30m
            memory: 80Mi
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
          mountPropagation: HostToContainer
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    app: node-exporter
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
  selector:
    app: node-exporter
EOF

# Aplicar
kubectl apply -f node-exporter-daemonset.yaml

# Verificar
kubectl get daemonset -n kube-system node-exporter
kubectl get pods -n kube-system -l app=node-exporter -o wide
```

---

## Configuración de Prometheus

### Archivo de Configuración Completo

```bash
sudo nano /etc/prometheus/prometheus.yml
```

**Contenido:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # kube-state-metrics (estado del cluster)
  - job_name: 'kube-state-metrics'
    metrics_path: '/api/v1/namespaces/kube-system/services/kube-state-metrics:8080/proxy/metrics'
    static_configs:
      - targets: ['localhost:8001']

  # API Server de Kubernetes
  - job_name: 'kubernetes-apiservers'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:8001']

  # Métricas de nodos (ajustar nombres según tus nodos)
  - job_name: 'kubernetes-nodes-prod'
    metrics_path: '/api/v1/nodes/gke-gke-prod-cluster-prod-32f0b1fc-9pfm/proxy/metrics'
    static_configs:
      - targets: ['localhost:8001']
        labels:
          node: 'gke-gke-prod-cluster-prod-32f0b1fc-9pfm'

  - job_name: 'kubernetes-nodes-system'
    metrics_path: '/api/v1/nodes/gke-gke-prod-cluster-system-a120926d-n8nv/proxy/metrics'
    static_configs:
      - targets: ['localhost:8001']
        labels:
          node: 'gke-gke-prod-cluster-system-a120926d-n8nv'

  # cAdvisor (uso de recursos de contenedores)
  - job_name: 'kubernetes-cadvisor-prod'
    metrics_path: '/api/v1/nodes/gke-gke-prod-cluster-prod-32f0b1fc-9pfm/proxy/metrics/cadvisor'
    static_configs:
      - targets: ['localhost:8001']
        labels:
          node: 'gke-gke-prod-cluster-prod-32f0b1fc-9pfm'

  - job_name: 'kubernetes-cadvisor-system'
    metrics_path: '/api/v1/nodes/gke-gke-prod-cluster-system-a120926d-n8nv/proxy/metrics/cadvisor'
    static_configs:
      - targets: ['localhost:8001']
        labels:
          node: 'gke-gke-prod-cluster-system-a120926d-n8nv'

  # node-exporter (métricas del SO)
  - job_name: 'node-exporter'
    metrics_path: '/api/v1/namespaces/kube-system/services/node-exporter:9100/proxy/metrics'
    static_configs:
      - targets: ['localhost:8001']
```

**Validar y reiniciar:**

```bash
# Validar configuración
promtool check config /etc/prometheus/prometheus.yml

# Reiniciar Prometheus
sudo systemctl restart prometheus

# Verificar targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

---

## Instalación y Configuración de Grafana

### 1. Instalar Grafana

```bash
# Agregar repositorio
sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | \
  sudo tee -a /etc/apt/sources.list.d/grafana.list

# Instalar
sudo apt-get update
sudo apt-get install -y grafana

# Iniciar y habilitar
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Crear regla de firewall
gcloud compute firewall-rules create allow-grafana \
  --project tfg-prod-478914 \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:3000 \
  --source-ranges=0.0.0.0/0
```

### 2. Configurar Grafana

**Acceso inicial:**
- URL: `http://34.41.209.157:3000`
- Usuario: `admin`
- Password: `admin` (cambiar en primer login)

**Agregar Prometheus como Data Source:**
1. Ir a Configuration → Data Sources
2. Add data source → Prometheus
3. URL: `http://localhost:9090`
4. Save & Test

**Importar Dashboards:**
1. Ir a Dashboards → Import
2. Ingresar IDs de dashboards recomendados:
   - **7249** - Kubernetes Cluster Monitoring
   - **6417** - Kubernetes Pods
   - **13332** - kube-state-metrics v2
3. Seleccionar Prometheus como data source
4. Import

---

## Métricas Disponibles

### kube-state-metrics
- Estado de pods, deployments, nodes
- Recursos solicitados vs límites
- Información de services y endpoints

### node-exporter
- CPU, memoria, disco, red del SO
- Procesos, file descriptors, load average

### cAdvisor
- Uso de CPU y memoria por contenedor
- I/O de disco y red por contenedor
- Throttling de recursos

---

## Queries PromQL Útiles

### Estado del Cluster

```promql
# Pods por estado
kube_pod_status_phase

# Pods running
kube_pod_status_phase{phase="Running"}

# Nodos disponibles
kube_node_status_condition{condition="Ready",status="true"}

# Pods en cada nodo
count by (node) (kube_pod_info)
```

### Recursos

```promql
# Uso de CPU de contenedores
rate(container_cpu_usage_seconds_total[5m])

# Uso de memoria de contenedores
container_memory_usage_bytes

# Uso de CPU por nodo
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memoria disponible por nodo
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

### Deployments

```promql
# Replicas disponibles vs deseadas
kube_deployment_status_replicas_available / kube_deployment_spec_replicas * 100

# Deployments con réplicas insuficientes
kube_deployment_status_replicas_available < kube_deployment_spec_replicas
```

---

## Comandos de Gestión

### Servicios systemd

```bash
# Ver estado
sudo systemctl status prometheus
sudo systemctl status grafana-server
sudo systemctl status kubectl-proxy

# Reiniciar
sudo systemctl restart prometheus
sudo systemctl restart grafana-server
sudo systemctl restart kubectl-proxy

# Ver logs
sudo journalctl -u prometheus -n 50
sudo journalctl -u grafana-server -f
sudo journalctl -u kubectl-proxy -f
```

### Kubernetes

```bash
# Ver pods de monitoreo
kubectl get pods -n kube-system | grep -E "kube-state-metrics|node-exporter"

# Ver estado del cluster
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory

# Ver logs
kubectl logs -n kube-system -l app=node-exporter --tail=50
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-state-metrics
```

### Verificación de Prometheus

```bash
# Ver todos los targets y su estado
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Probar queries
curl -s "http://localhost:9090/api/v1/query?query=up" | jq
curl -s "http://localhost:9090/api/v1/query?query=node_cpu_seconds_total" | jq '.data.result | length'
```

---

## Troubleshooting

### Problema: No puedo acceder a Grafana/Prometheus

```bash
# Verificar servicios
sudo systemctl status grafana-server
sudo systemctl status prometheus

# Verificar reglas de firewall
gcloud compute firewall-rules list --project tfg-prod-478914 | grep -E "grafana|prometheus"

# Verificar puertos
sudo netstat -tulpn | grep -E "3000|9090"
```

### Problema: Targets DOWN en Prometheus

```bash
# Verificar kubectl proxy
sudo systemctl status kubectl-proxy

# Probar acceso manual
curl http://localhost:8001/api/v1/namespaces/kube-system/services/kube-state-metrics:8080/proxy/metrics | head -20

# Reiniciar servicios
sudo systemctl restart kubectl-proxy
sudo systemctl restart prometheus
```

### Problema: Pods en Pending

```bash
# Ver detalles del problema
kubectl describe pod -n kube-system <pod-name>

# Si es por taints, aplicar tolerations (ver secciones anteriores)
# Si es por recursos, reducir requests en el manifiesto
```

### Problema: Dashboards muestran "No data"

```bash
# Verificar targets en Prometheus
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health != "up")'

# Verificar data source en Grafana
# Settings → Data Sources → Prometheus → Test

# Verificar que Prometheus tiene datos
curl -s "http://localhost:9090/api/v1/query?query=up" | jq
```

---

## Próximos Pasos

1. **Alertmanager:** Configurar alertas y notificaciones (email, Slack, PagerDuty)
2. **Dashboards personalizados:** Crear visualizaciones específicas para tus aplicaciones
3. **Retención de datos:** Ajustar según necesidades (default 15 días)
4. **Alta disponibilidad:** Múltiples instancias de Prometheus, Thanos para almacenamiento largo plazo
5. **Exporters adicionales:** Blackbox exporter para endpoints, custom exporters para aplicaciones

---

## Accesos Rápidos

- **Prometheus:** `http://34.41.209.157:9090`
- **Grafana:** `http://34.41.209.157:3000` (admin/admin)
- **SSH VM:** `gcloud compute ssh prod-monitoring-vm --zone us-central1-a --project tfg-prod-478914`
- **Configurar kubectl:** `gcloud container clusters get-credentials gke-prod-cluster --location us-central1-a --project tfg-prod-478914`

---

## Referencias

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
- [node-exporter](https://github.com/prometheus/node_exporter)
- [PromQL Query Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [GKE Monitoring Best Practices](https://cloud.google.com/kubernetes-engine/docs/how-to/monitoring)

---

**Documentación:** Noviembre 2025  
**Proyecto:** TFG Prod - Monitoreo GKE
