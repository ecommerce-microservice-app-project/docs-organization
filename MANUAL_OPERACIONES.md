# Manual de Operaciones - Sistema E-commerce

## 1. Introducción

Este manual proporciona los procedimientos y comandos necesarios para operar y mantener el sistema de e-commerce basado en microservicios. Está dirigido a operadores, administradores de sistemas y desarrolladores que necesitan realizar tareas de operación en los ambientes de desarrollo, staging y producción.

### 1.1 Alcance

Este manual cubre:

- Acceso y configuración de clusters Kubernetes
- Comandos básicos de operación
- Procedimientos de despliegue y actualización
- Monitoreo y verificación de salud
- Troubleshooting común
- Escalado de servicios
- Backup y recuperación
- Mantenimiento programado

### 1.2 Arquitectura del Sistema

El sistema está compuesto por:

- **10 microservicios** desplegados en Kubernetes
- **3 ambientes**: Development (Azure), Staging (Azure), Production (GCP)
- **Service Discovery**: Eureka para registro y descubrimiento de servicios
- **API Gateway**: Punto de entrada único con TLS/HTTPS
- **CI/CD**: GitHub Actions para despliegue automático

**Microservicios de Infraestructura:**
- Service Discovery (Eureka)
- Cloud Config (opcional)
- API Gateway
- Proxy Client

**Microservicios de Negocio:**
- Product Service
- Order Service
- User Service
- Shipping Service
- Payment Service
- Favourite Service

## 2. Acceso a los Clusters

### 2.1 Azure Development (AKS)

**Cluster**: `az-k8s-cluster`  
**Región**: East US 2  
**Resource Group**: Configurado en secrets de GitHub  
**Namespace**: `ecommerce-dev`

**Autenticación:**
```bash
# Login a Azure
az login

# Obtener credenciales del cluster
az aks get-credentials \
  --resource-group <RESOURCE_GROUP> \
  --name az-k8s-cluster \
  --overwrite-existing

# Verificar conexión
kubectl cluster-info
kubectl get nodes
```

### 2.2 Azure Staging (AKS)

**Cluster**: `az-k8s-stage-cluster`  
**Región**: East US 2  
**Resource Group**: Configurado en secrets de GitHub  
**Namespace**: `ecommerce-dev`

**Autenticación:**
```bash
# Login a Azure
az login

# Obtener credenciales del cluster
az aks get-credentials \
  --resource-group <RESOURCE_GROUP> \
  --name az-k8s-stage-cluster \
  --overwrite-existing

# Verificar conexión
kubectl cluster-info
kubectl get nodes
```

### 2.3 GCP Production (GKE)

**Cluster**: `gke-prod-cluster`  
**Región**: us-central1-a  
**Proyecto**: Configurado en secrets de GitHub  
**Namespace**: `ecommerce-dev`

**Autenticación:**
```bash
# Login a GCP
gcloud auth login

# Configurar proyecto
gcloud config set project <PROJECT_ID>

# Obtener credenciales del cluster
gcloud container clusters get-credentials gke-prod-cluster \
  --zone us-central1-a \
  --project <PROJECT_ID>

# Verificar conexión
kubectl cluster-info
kubectl get nodes
```

**Nota**: Para producción, se requiere autenticación con Service Account. Las credenciales están configuradas en GitHub Secrets (`GCP_CREDENTIALS_PROD`).

## 3. Comandos Básicos de Operación

### 3.1 Verificación del Estado del Sistema

**Ver todos los pods:**
```bash
kubectl get pods -n ecommerce-dev
```

**Ver pods de un servicio específico:**
```bash
kubectl get pods -n ecommerce-dev -l app=product-service
```

**Ver servicios:**
```bash
kubectl get svc -n ecommerce-dev
```

**Ver deployments:**
```bash
kubectl get deployments -n ecommerce-dev
```

**Ver estado detallado de un pod:**
```bash
kubectl describe pod <POD_NAME> -n ecommerce-dev
```

**Ver logs de un pod:**
```bash
# Logs en tiempo real
kubectl logs -f <POD_NAME> -n ecommerce-dev

# Logs de un deployment
kubectl logs -f deployment/product-service -n ecommerce-dev

# Logs de los últimos 100 líneas
kubectl logs --tail=100 <POD_NAME> -n ecommerce-dev

# Logs de todos los pods con un label
kubectl logs -f -l app=product-service -n ecommerce-dev
```

### 3.2 Verificación de Health Checks

**Verificar health check de un servicio:**
```bash
# Obtener IP del servicio
kubectl get svc api-gateway -n ecommerce-dev

# Port-forward para acceso local
kubectl port-forward svc/api-gateway 8080:8080 -n ecommerce-dev

# Verificar health check
curl http://localhost:8080/actuator/health
```

**Health checks disponibles:**
- API Gateway: `http://<IP>:8080/actuator/health`
- Product Service: `http://<IP>:8500/product-service/actuator/health`
- Order Service: `http://<IP>:8084/order-service/actuator/health`
- User Service: `http://<IP>:8083/user-service/actuator/health`
- Shipping Service: `http://<IP>:8600/shipping-service/actuator/health`
- Payment Service: `http://<IP>:8400/payment-service/actuator/health`
- Favourite Service: `http://<IP>:8800/favourite-service/actuator/health`

**Service Discovery (Eureka Dashboard):**
```bash
# Port-forward
kubectl port-forward svc/service-discovery 8761:8761 -n ecommerce-dev

# Acceder en navegador
# http://localhost:8761
```

### 3.3 Verificación de Recursos

**Ver uso de recursos:**
```bash
# Top de pods por CPU
kubectl top pods -n ecommerce-dev

# Top de pods por memoria
kubectl top pods -n ecommerce-dev --sort-by=memory

# Ver recursos solicitados y límites
kubectl describe pod <POD_NAME> -n ecommerce-dev | grep -A 5 "Limits\|Requests"
```

## 4. Despliegue y Actualización

### 4.1 Despliegue Automático (CI/CD)

Los despliegues se realizan automáticamente mediante GitHub Actions cuando se hace push a las ramas correspondientes:

- **Dev**: Push a `dev` o `develop` → Tag `dev-latest`
- **Stage**: Push a `stage` → Tag `stage-latest`
- **Prod**: Push a `main` o `master` → Tag `prod-0.1.0` (con versionado semántico)

**Verificar despliegue en pipeline:**
1. Ir a GitHub Actions del repositorio
2. Verificar que el job `deploy` completó exitosamente
3. Verificar logs del despliegue

### 4.2 Despliegue Manual

**Actualizar imagen de un servicio:**
```bash
# Opción 1: Actualizar imagen en deployment
kubectl set image deployment/product-service \
  product-service=docker.io/juanc7773/product-service:dev-latest \
  -n ecommerce-dev

# Opción 2: Editar deployment directamente
kubectl edit deployment product-service -n ecommerce-dev
```

**Forzar rollout restart (para tags estáticos):**
```bash
kubectl rollout restart deployment/product-service -n ecommerce-dev
```

**Verificar estado del rollout:**
```bash
kubectl rollout status deployment/product-service -n ecommerce-dev
```

**Rollback a versión anterior:**
```bash
# Ver historial de rollouts
kubectl rollout history deployment/product-service -n ecommerce-dev

# Rollback a versión anterior
kubectl rollout undo deployment/product-service -n ecommerce-dev

# Rollback a versión específica
kubectl rollout undo deployment/product-service --to-revision=2 -n ecommerce-dev
```

### 4.3 Orden de Despliegue

Para despliegues manuales completos, seguir este orden:

1. **Service Discovery** (debe iniciar primero)
2. **Cloud Config** (opcional)
3. **Microservicios de Negocio** (Product, Order, User, Shipping, Payment, Favourite)
4. **API Gateway**
5. **Proxy Client**

**Script de ejemplo:**
```bash
# 1. Service Discovery
kubectl apply -f k8s/service-discovery/deployment.yaml -n ecommerce-dev
kubectl wait --for=condition=ready pod -l app=service-discovery -n ecommerce-dev --timeout=300s

# 2. Cloud Config (opcional)
kubectl apply -f k8s/cloud-config/deployment.yaml -n ecommerce-dev

# 3. Microservicios de negocio
kubectl apply -f k8s/product-service/deployment.yaml -n ecommerce-dev
kubectl apply -f k8s/order-service/deployment.yaml -n ecommerce-dev
kubectl apply -f k8s/user-service/deployment.yaml -n ecommerce-dev
kubectl apply -f k8s/shipping-service/deployment.yaml -n ecommerce-dev
kubectl apply -f k8s/payment-service/deployment.yaml -n ecommerce-dev
kubectl apply -f k8s/favourite-service/deployment.yaml -n ecommerce-dev

# 4. API Gateway
kubectl apply -f k8s/api-gateway/configmap.yaml -n ecommerce-dev
kubectl apply -f k8s/api-gateway/deployment.yaml -n ecommerce-dev

# 5. Proxy Client
kubectl apply -f k8s/proxy-client/deployment.yaml -n ecommerce-dev
```

## 5. Monitoreo y Verificación

### 5.1 Verificación de Servicios Registrados en Eureka

**Acceder al dashboard de Eureka:**
```bash
# Port-forward
kubectl port-forward svc/service-discovery 8761:8761 -n ecommerce-dev

# Abrir en navegador: http://localhost:8761
```

**Verificar servicios registrados vía API:**
```bash
curl http://localhost:8761/eureka/apps
```

### 5.2 Verificación de Endpoints del API Gateway

**Endpoints disponibles:**
- Product Service: `http://<API_GATEWAY_IP>/product-service/api/products`
- Order Service: `http://<API_GATEWAY_IP>/order-service/api/orders`
- User Service: `http://<API_GATEWAY_IP>/user-service/api/users`
- Payment Service: `http://<API_GATEWAY_IP>/payment-service/api/payments`
- Shipping Service: `http://<API_GATEWAY_IP>/shipping-service/api/shippings`
- Favourite Service: `http://<API_GATEWAY_IP>/favourite-service/api/favourites`
- Proxy Client: `http://<API_GATEWAY_IP>/app/**`

**En producción (HTTPS):**
- Dominio: `https://api.alianzadelamagiaeterna.com`
- Ejemplo: `https://api.alianzadelamagiaeterna.com/product-service/api/products`

### 5.3 Verificación de Certificados TLS (Producción)

**Verificar certificado del Ingress:**
```bash
# Ver certificado obtenido por Cert-Manager
kubectl get certificate -n ecommerce-dev

# Ver detalles del certificado
kubectl describe certificate api-gateway-tls -n ecommerce-dev

# Verificar certificado desde el navegador
# https://api.alianzadelamagiaeterna.com
```

**Verificar ClusterIssuer de Cert-Manager:**
```bash
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

## 6. Troubleshooting

### 6.1 Pods en Estado Pending

**Causa común**: Recursos insuficientes o taints sin tolerations.

**Diagnóstico:**
```bash
kubectl describe pod <POD_NAME> -n ecommerce-dev
```

**Solución:**
- Verificar que hay nodos disponibles: `kubectl get nodes`
- Verificar taints en nodos: `kubectl describe node <NODE_NAME>`
- Verificar que el deployment tiene tolerations configuradas

### 6.2 Pods en Estado CrashLoopBackOff

**Diagnóstico:**
```bash
# Ver logs del pod
kubectl logs <POD_NAME> -n ecommerce-dev

# Ver eventos
kubectl describe pod <POD_NAME> -n ecommerce-dev

# Ver logs de contenedor anterior (si se reinició)
kubectl logs <POD_NAME> --previous -n ecommerce-dev
```

**Causas comunes:**
- Error de configuración (variables de entorno incorrectas)
- Error de conexión a base de datos
- Error de conexión a Service Discovery
- Out of Memory (OOM)

**Solución:**
- Revisar logs para identificar el error específico
- Verificar ConfigMaps y Secrets
- Verificar conectividad a base de datos
- Verificar que Service Discovery está disponible

### 6.3 Servicio No Responde

**Diagnóstico:**
```bash
# Verificar que el pod está corriendo
kubectl get pods -l app=<SERVICE_NAME> -n ecommerce-dev

# Verificar que el servicio está configurado
kubectl get svc <SERVICE_NAME> -n ecommerce-dev

# Verificar endpoints del servicio
kubectl get endpoints <SERVICE_NAME> -n ecommerce-dev

# Probar conectividad desde dentro del cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Dentro del pod: wget -O- http://<SERVICE_NAME>:<PORT>/actuator/health
```

**Solución:**
- Verificar que el pod está en estado `Running`
- Verificar que el servicio tiene endpoints asignados
- Verificar health checks del servicio
- Verificar logs del servicio

### 6.4 Error de Conexión a Service Discovery

**Diagnóstico:**
```bash
# Verificar que Service Discovery está corriendo
kubectl get pods -l app=service-discovery -n ecommerce-dev

# Verificar logs de Service Discovery
kubectl logs -l app=service-discovery -n ecommerce-dev

# Verificar que el servicio puede alcanzar Service Discovery
kubectl exec <POD_NAME> -n ecommerce-dev -- wget -O- http://service-discovery:8761/eureka/apps
```

**Solución:**
- Asegurar que Service Discovery está desplegado y corriendo
- Verificar configuración de Eureka en el servicio (URL correcta)
- Verificar conectividad de red entre pods

### 6.5 Error de Certificado TLS (Producción)

**Diagnóstico:**
```bash
# Verificar certificado
kubectl get certificate -n ecommerce-dev

# Verificar desafío de Let's Encrypt
kubectl get challenges -n ecommerce-dev

# Ver logs de Cert-Manager
kubectl logs -n cert-manager -l app=cert-manager
```

**Solución:**
- Verificar que el DNS apunta correctamente al Ingress Controller
- Verificar que el Ingress tiene la anotación correcta de Cert-Manager
- Verificar que el ClusterIssuer está configurado correctamente
- Esperar a que Cert-Manager complete el desafío (puede tomar varios minutos)

### 6.6 Pods con Alto Uso de Recursos

**Diagnóstico:**
```bash
# Ver uso actual de recursos
kubectl top pods -n ecommerce-dev

# Ver límites configurados
kubectl describe pod <POD_NAME> -n ecommerce-dev | grep -A 5 "Limits\|Requests"
```

**Solución:**
- Aumentar límites de recursos en el deployment si es necesario
- Escalar horizontalmente (aumentar réplicas)
- Investigar memory leaks o problemas de rendimiento en la aplicación

## 7. Escalado

### 7.1 Escalado Horizontal (Réplicas)

**Aumentar réplicas de un servicio:**
```bash
kubectl scale deployment/product-service --replicas=3 -n ecommerce-dev
```

**Verificar escalado:**
```bash
kubectl get pods -l app=product-service -n ecommerce-dev
kubectl get deployment product-service -n ecommerce-dev
```

**Reducir réplicas:**
```bash
kubectl scale deployment/product-service --replicas=1 -n ecommerce-dev
```

### 7.2 Escalado Vertical (Recursos)

**Editar recursos de un deployment:**
```bash
kubectl edit deployment/product-service -n ecommerce-dev
```

**Buscar la sección de recursos y modificar:**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi
```

**Aplicar cambios:**
```bash
kubectl apply -f deployment.yaml -n ecommerce-dev
```

### 7.3 Auto-scaling (HPA)

**Crear HorizontalPodAutoscaler:**
```bash
kubectl autoscale deployment/product-service \
  --cpu-percent=80 \
  --min=1 \
  --max=5 \
  -n ecommerce-dev
```

**Verificar HPA:**
```bash
kubectl get hpa -n ecommerce-dev
kubectl describe hpa product-service -n ecommerce-dev
```

## 8. Backup y Recuperación

### 8.1 Backup de Configuraciones

**Exportar deployments:**
```bash
kubectl get deployment -n ecommerce-dev -o yaml > backups/deployments-$(date +%Y%m%d).yaml
```

**Exportar ConfigMaps:**
```bash
kubectl get configmap -n ecommerce-dev -o yaml > backups/configmaps-$(date +%Y%m%d).yaml
```

**Exportar Secrets (cuidado con datos sensibles):**
```bash
kubectl get secret -n ecommerce-dev -o yaml > backups/secrets-$(date +%Y%m%d).yaml
```

### 8.2 Backup de Bases de Datos

**Nota**: Las bases de datos están fuera del scope de Kubernetes. Consultar documentación específica de cada base de datos (MySQL, PostgreSQL) para procedimientos de backup.

**Ejemplo para PostgreSQL (Order Service, User Service):**
```bash
# Desde un pod con acceso a la base de datos
kubectl exec -it <POSTGRES_POD> -n ecommerce-dev -- pg_dump -U <USER> <DATABASE> > backup-$(date +%Y%m%d).sql
```

**Ejemplo para MySQL (Product Service, Payment Service, Shipping Service, Favourite Service):**
```bash
# Desde un pod con acceso a la base de datos
kubectl exec -it <MYSQL_POD> -n ecommerce-dev -- mysqldump -u <USER> -p <DATABASE> > backup-$(date +%Y%m%d).sql
```

### 8.3 Recuperación

**Restaurar deployment:**
```bash
kubectl apply -f backups/deployments-YYYYMMDD.yaml -n ecommerce-dev
```

**Restaurar ConfigMap:**
```bash
kubectl apply -f backups/configmaps-YYYYMMDD.yaml -n ecommerce-dev
```

**Restaurar base de datos:**
```bash
# PostgreSQL
kubectl exec -i <POSTGRES_POD> -n ecommerce-dev -- psql -U <USER> <DATABASE> < backup-YYYYMMDD.sql

# MySQL
kubectl exec -i <MYSQL_POD> -n ecommerce-dev -- mysql -u <USER> -p <DATABASE> < backup-YYYYMMDD.sql
```

## 9. Mantenimiento

### 9.1 Actualización de Imágenes

**Proceso recomendado:**
1. Verificar que la nueva imagen está disponible en Docker Hub
2. Actualizar el deployment con la nueva imagen
3. Monitorear el rollout
4. Verificar health checks después del despliegue

**Comando:**
```bash
kubectl set image deployment/<SERVICE_NAME> \
  <SERVICE_NAME>=docker.io/juanc7773/<SERVICE_NAME>:<NEW_TAG> \
  -n ecommerce-dev
```

### 9.2 Limpieza de Recursos

**Eliminar pods en estado Error:**
```bash
kubectl delete pod <POD_NAME> -n ecommerce-dev
```

**Limpiar ReplicaSets antiguos:**
```bash
# Los deployments tienen revisionHistoryLimit: 2 configurado
# Los ReplicaSets antiguos se eliminan automáticamente
# Para limpiar manualmente:
kubectl delete replicaset <OLD_REPLICASET_NAME> -n ecommerce-dev
```

**Limpiar imágenes no utilizadas (en nodos):**
```bash
# Ejecutar en cada nodo (requiere acceso SSH)
docker image prune -a
```

### 9.3 Actualización de Certificados TLS

Los certificados se renuevan automáticamente mediante Cert-Manager. Para verificar:

```bash
# Ver fecha de expiración del certificado
kubectl get certificate api-gateway-tls -n ecommerce-dev -o jsonpath='{.status.notAfter}'

# Verificar renovación automática
kubectl describe certificate api-gateway-tls -n ecommerce-dev
```

### 9.4 Mantenimiento Programado

**Recomendaciones:**
- Revisar logs semanalmente para detectar problemas
- Verificar uso de recursos mensualmente
- Actualizar imágenes de contenedores cuando haya actualizaciones de seguridad
- Revisar y limpiar recursos no utilizados trimestralmente

## 10. Comandos de Referencia Rápida

### 10.1 Verificación Rápida del Sistema

```bash
# Estado general
kubectl get all -n ecommerce-dev

# Pods por servicio
kubectl get pods -n ecommerce-dev -l app=product-service
kubectl get pods -n ecommerce-dev -l app=order-service
kubectl get pods -n ecommerce-dev -l app=user-service

# Servicios
kubectl get svc -n ecommerce-dev

# Deployments
kubectl get deployments -n ecommerce-dev
```

### 10.2 Logs Rápidos

```bash
# Logs de un servicio
kubectl logs -f deployment/product-service -n ecommerce-dev

# Logs de todos los pods de un servicio
kubectl logs -f -l app=product-service -n ecommerce-dev

# Logs de los últimos 50 líneas
kubectl logs --tail=50 deployment/product-service -n ecommerce-dev
```

### 10.3 Port-Forward Rápido

```bash
# Service Discovery
kubectl port-forward svc/service-discovery 8761:8761 -n ecommerce-dev

# API Gateway
kubectl port-forward svc/api-gateway 8080:8080 -n ecommerce-dev

# Product Service
kubectl port-forward svc/product-service 8500:8500 -n ecommerce-dev
```

### 10.4 Rollout y Rollback

```bash
# Ver historial
kubectl rollout history deployment/product-service -n ecommerce-dev

# Rollback
kubectl rollout undo deployment/product-service -n ecommerce-dev

# Ver estado
kubectl rollout status deployment/product-service -n ecommerce-dev
```

## 11. Contactos y Responsabilidades

### 11.1 Equipo de Operaciones

- **Despliegues**: GitHub Actions (automático) o equipo de DevOps
- **Troubleshooting**: Equipo de desarrollo y operaciones
- **Mantenimiento**: Equipo de operaciones

### 11.2 Recursos Adicionales

- **Documentación de Servicios**: Ver `docs-organization/SERVICIOS.md`
- **Costos de Infraestructura**: Ver `docs-organization/COSTOS_INFRAESTRUCTURA.md`
- **Repositorio de Infraestructura**: `kubernetes-organization`
- **Pipelines CI/CD**: GitHub Actions en cada repositorio de servicio

### 11.3 URLs Importantes

**Producción:**
- API Gateway (HTTPS): `https://api.alianzadelamagiaeterna.com`
- Service Discovery Dashboard: Port-forward a `http://localhost:8761`

**Development:**
- API Gateway: `http://68.154.28.85:8080`

**Staging:**
- API Gateway: `http://52.167.161.77:8080`

## 12. Anexos

### 12.1 Especificaciones de Recursos por Servicio

| Servicio | Requests CPU | Requests RAM | Limits CPU | Limits RAM |
|----------|-------------|--------------|------------|------------|
| Service Discovery | 50m | 128Mi | 500m | 512Mi |
| Cloud Config | 50m | 128Mi | 500m | 512Mi |
| API Gateway | 100m | 256Mi | 500m | 512Mi |
| Product Service | 50m | 128Mi | 500m | 512Mi |
| Order Service | 50m | 128Mi | 500m | 512Mi |
| User Service | 25m | 64Mi | 500m | 512Mi |
| Shipping Service | 25m | 64Mi | 500m | 512Mi |
| Payment Service | 25m | 64Mi | 500m | 512Mi |
| Favourite Service | 50m | 128Mi | 500m | 512Mi |
| Proxy Client | 100m | 256Mi | 500m | 512Mi |

### 12.2 Puertos por Servicio

| Servicio | Puerto Interno | Puerto Externo |
|----------|----------------|----------------|
| Service Discovery | 8761 | Port-forward |
| Cloud Config | 9296 | Port-forward |
| API Gateway | 8080 | LoadBalancer/Ingress |
| Product Service | 8500 | A través de API Gateway |
| Order Service | 8084 | A través de API Gateway |
| User Service | 8083 | A través de API Gateway |
| Shipping Service | 8600 | A través de API Gateway |
| Payment Service | 8400 | A través de API Gateway |
| Favourite Service | 8800 | A través de API Gateway |
| Proxy Client | 8900 | A través de API Gateway |

### 12.3 Tags de Imagen por Ambiente

| Ambiente | Tag | Branch |
|----------|-----|--------|
| Development | `dev-latest` | `dev`, `develop` |
| Staging | `stage-latest` | `stage` |
| Production | `prod-0.1.0` (semántico) | `main`, `master` |

---

**Última actualización**: 2025-26-11  
**Versión del Manual**: 1.0

