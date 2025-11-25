# Planes de Rollback

## 1. Introducción

Este documento describe los procedimientos para realizar rollback (reversión) de despliegues en los diferentes ambientes del ecommerce. Un rollback es necesario cuando un despliegue causa problemas críticos en producción.

## 2. Estrategias de Rollback

### 2.1 Rollback por Versión (Recomendado)

Revertir a una versión anterior específica que se sabe que funcionaba correctamente.

### 2.2 Rollback por Imagen Docker

Revertir a una imagen Docker específica usando su tag o SHA.

### 2.3 Rollback por Configuración

Revertir cambios en ConfigMaps o Secrets que causaron problemas.

## 3. Rollback en Desarrollo (Dev)

### 3.1 Rollback Manual

```bash
# 1. Conectar al cluster de desarrollo
az aks get-credentials --resource-group <RESOURCE_GROUP_DEV> --name <CLUSTER_NAME_DEV>

# 2. Listar deployments disponibles
kubectl get deployments -n ecommerce-dev

# 3. Ver historial de rollouts
kubectl rollout history deployment/<SERVICE_NAME> -n ecommerce-dev

# 4. Rollback a versión anterior
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-dev

# 5. Verificar rollback
kubectl rollout status deployment/<SERVICE_NAME> -n ecommerce-dev
```

### 3.2 Rollback a Versión Específica

```bash
# 1. Ver historial con revisiones
kubectl rollout history deployment/<SERVICE_NAME> -n ecommerce-dev

# 2. Rollback a revisión específica (ej: revisión 3)
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-dev --to-revision=3
```

### 3.3 Rollback por Imagen Docker

```bash
# 1. Editar deployment
kubectl set image deployment/<SERVICE_NAME> \
  <CONTAINER_NAME>=<REGISTRY>/<IMAGE_NAME>:<TAG_ANTERIOR> \
  -n ecommerce-dev

# 2. Verificar
kubectl rollout status deployment/<SERVICE_NAME> -n ecommerce-dev
```

## 4. Rollback en Stage

### 4.1 Rollback Manual

```bash
# 1. Conectar al cluster de stage
az aks get-credentials --resource-group <RESOURCE_GROUP_STAGE> --name <CLUSTER_NAME_STAGE>

# 2. Rollback
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-stage

# 3. Verificar
kubectl rollout status deployment/<SERVICE_NAME> -n ecommerce-stage
```

### 4.2 Rollback a Versión Específica

```bash
# Ver historial
kubectl rollout history deployment/<SERVICE_NAME> -n ecommerce-stage

# Rollback a revisión específica
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-stage --to-revision=<REVISION>
```

## 5. Rollback en Producción

### 5.1 Procedimiento de Emergencia

**⚠️ IMPORTANTE: Rollback en producción debe hacerse con cuidado y documentación**

#### Paso 1: Identificar el Problema

```bash
# Ver logs del pod actual
kubectl logs -f deployment/<SERVICE_NAME> -n ecommerce-prod

# Ver eventos
kubectl get events -n ecommerce-prod --sort-by='.lastTimestamp'

# Verificar health checks
kubectl get pods -n ecommerce-prod -l app=<SERVICE_NAME>
```

#### Paso 2: Conectar al Cluster de Producción

```bash
# GKE Production
gcloud container clusters get-credentials <CLUSTER_NAME_PROD> \
  --location <LOCATION> \
  --project <PROJECT_ID>
```

#### Paso 3: Identificar Versión Anterior

```bash
# Ver historial de rollouts
kubectl rollout history deployment/<SERVICE_NAME> -n ecommerce-prod

# Ver tags de imágenes disponibles en Docker Hub
# O revisar GitHub Releases para ver versiones anteriores
```

#### Paso 4: Ejecutar Rollback

**Opción A: Rollback a revisión anterior**
```bash
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-prod
kubectl rollout status deployment/<SERVICE_NAME> -n ecommerce-prod
```

**Opción B: Rollback a versión específica**
```bash
# Ver historial con detalles
kubectl rollout history deployment/<SERVICE_NAME> -n ecommerce-prod --revision=<REVISION>

# Rollback a revisión específica
kubectl rollout undo deployment/<SERVICE_NAME> -n ecommerce-prod --to-revision=<REVISION>
```

**Opción C: Rollback por imagen Docker específica**
```bash
# Editar deployment con imagen anterior
kubectl set image deployment/<SERVICE_NAME> \
  <CONTAINER_NAME>=<REGISTRY>/<IMAGE_NAME>:<VERSION_ANTERIOR> \
  -n ecommerce-prod

# Forzar rollout restart
kubectl rollout restart deployment/<SERVICE_NAME> -n ecommerce-prod

# Verificar
kubectl rollout status deployment/<SERVICE_NAME> -n ecommerce-prod
```

#### Paso 5: Verificar Rollback Exitoso

```bash
# Verificar pods
kubectl get pods -n ecommerce-prod -l app=<SERVICE_NAME>

# Verificar logs
kubectl logs -f deployment/<SERVICE_NAME> -n ecommerce-prod

# Verificar health endpoint
curl https://<API_GATEWAY_URL>/<SERVICE_NAME>/actuator/health

# Verificar métricas
kubectl top pods -n ecommerce-prod
```

### 5.2 Rollback por Servicio

#### API Gateway
```bash
kubectl rollout undo deployment/api-gateway -n ecommerce-prod
```

#### Product Service
```bash
kubectl rollout undo deployment/product-service -n ecommerce-prod
```

#### Order Service
```bash
kubectl rollout undo deployment/order-service -n ecommerce-prod
```

#### User Service
```bash
kubectl rollout undo deployment/user-service -n ecommerce-prod
```

#### Payment Service
```bash
kubectl rollout undo deployment/payment-service -n ecommerce-prod
```

#### Favourite Service
```bash
kubectl rollout undo deployment/favourite-service -n ecommerce-prod
```

#### Shipping Service
```bash
kubectl rollout undo deployment/shipping-service -n ecommerce-prod
```

## 6. Rollback de ConfigMaps y Secrets

### 6.1 Ver Historial de ConfigMap

```bash
# Ver versiones anteriores (si están versionadas)
kubectl get configmap <CONFIGMAP_NAME> -n <NAMESPACE> -o yaml

# Editar y restaurar configuración anterior
kubectl edit configmap <CONFIGMAP_NAME> -n <NAMESPACE>
```

### 6.2 Rollback de Secret

```bash
# Editar secret
kubectl edit secret <SECRET_NAME> -n <NAMESPACE>

# O aplicar desde archivo anterior
kubectl apply -f <SECRET_FILE_ANTERIOR>.yaml -n <NAMESPACE>
```

### 6.3 Reiniciar Pods Después de Cambiar ConfigMap/Secret

```bash
# Forzar restart para aplicar nueva configuración
kubectl rollout restart deployment/<SERVICE_NAME> -n <NAMESPACE>
```

## 7. Rollback Automatizado (Futuro)

### 7.1 Health Check Automático

Implementar health checks que automaticen rollback si:
- Health check falla por más de X minutos
- Error rate excede umbral
- Latencia excede umbral

### 7.2 Canary Deployments

Implementar canary deployments para detectar problemas antes del despliegue completo.

## 8. Documentación Post-Rollback

Después de un rollback, documentar:

1. **Razón del rollback**: ¿Qué problema causó el rollback?
2. **Versión revertida**: ¿A qué versión se hizo rollback?
3. **Versión problemática**: ¿Qué versión causó el problema?
4. **Acciones tomadas**: ¿Qué se hizo para resolver?
5. **Lecciones aprendidas**: ¿Qué se puede mejorar?

### Template de Documentación

```markdown
## Rollback - [Fecha]

**Servicio**: [Nombre del servicio]
**Versión Problemática**: [Versión que causó el problema]
**Versión de Rollback**: [Versión a la que se hizo rollback]
**Ambiente**: [dev/stage/prod]

**Problema Identificado**:
[Descripción del problema]

**Acciones Tomadas**:
1. [Acción 1]
2. [Acción 2]

**Resultado**:
[Estado después del rollback]

**Lecciones Aprendidas**:
[Qué se puede mejorar]
```

## 9. Prevención de Rollbacks

### 9.1 Mejores Prácticas

1. **Tests exhaustivos**: Asegurar que todos los tests pasen antes de merge
2. **Code review**: Revisión cuidadosa de cambios
3. **Staging primero**: Siempre desplegar a stage antes de prod
4. **Monitoreo**: Monitorear métricas después de cada despliegue
5. **Feature flags**: Usar feature flags para habilitar/deshabilitar features

### 9.2 Checklist Pre-Producción

- [ ] Todos los tests pasan
- [ ] Code review aprobado
- [ ] Desplegado y validado en stage
- [ ] Health checks funcionando
- [ ] Documentación actualizada
- [ ] Plan de rollback identificado

## 10. Contacto de Emergencia

Para rollbacks de emergencia en producción, contactar:
- **DevOps Team**: [Contacto]
- **On-Call Engineer**: [Contacto]

## 11. Recursos Adicionales

- [Kubernetes Rollout History](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
- [Change Management Process](./CHANGE_MANAGEMENT.md)
- [GitHub Releases](https://github.com/[ORG]/[REPO]/releases) - Ver versiones anteriores

