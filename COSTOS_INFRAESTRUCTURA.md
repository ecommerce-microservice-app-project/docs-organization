# Costos de Infraestructura

Este documento detalla los costos estimados de la infraestructura multi-cloud utilizada para el proyecto de ecommerce.

## Resumen Ejecutivo

El proyecto utiliza una arquitectura multi-cloud con tres ambientes desplegados en diferentes proveedores:

- **Azure Development**: Cuenta de estudiante Azure (primera cuenta)
- **Azure Staging**: Cuenta de estudiante Azure (segunda cuenta)
- **GCP Production**: Cuenta de prueba GCP con créditos

**Costo Total Estimado Mensual**: ~$110-130 USD/mes (sin créditos)

**Costo Real con Créditos**: $0 USD/mes (durante período de créditos)

## Arquitectura de Infraestructura

### Azure Development

**Proveedor**: Microsoft Azure (Cuenta de Estudiante - Primera cuenta)  
**Región**: East US 2  
**Cluster**: Azure Kubernetes Service (AKS)

**Recursos**:
- 1 nodo System Pool: Standard_B2s (2 vCPU, 2GB RAM)
- 1 nodo Production Pool: Standard_B2s (2 vCPU, 2GB RAM)
- Total: 2 nodos

**Costos Estimados**:
- Standard_B2s: ~$15-20 USD/nodo/mes
- 2 nodos: ~$30-40 USD/mes
- AKS Management: Gratis
- **Total Mensual**: ~$30-40 USD/mes

**Créditos Disponibles**: $100 USD de crédito de estudiante Azure

### Azure Staging

**Proveedor**: Microsoft Azure (Cuenta de Estudiante - Segunda cuenta)  
**Región**: East US 2  
**Cluster**: Azure Kubernetes Service (AKS)

**Recursos**:
- 1 nodo System Pool: Standard_B2s (2 vCPU, 2GB RAM)
- 1 nodo Production Pool: Standard_B2s (2 vCPU, 2GB RAM)
- Total: 2 nodos

**Costos Estimados**:
- Standard_B2s: ~$15-20 USD/nodo/mes
- 2 nodos: ~$30-40 USD/mes
- AKS Management: Gratis
- **Total Mensual**: ~$30-40 USD/mes

**Créditos Disponibles**: $100 USD de crédito de estudiante Azure (segunda cuenta)

### GCP Production

**Proveedor**: Google Cloud Platform (Cuenta de Prueba)  
**Región**: us-central1  
**Cluster**: Google Kubernetes Engine (GKE)

**Recursos**:
- 1 nodo System Pool: e2-medium (2 vCPU, 4GB RAM)
- 1 nodo Production Pool: e2-medium (2 vCPU, 4GB RAM)
- 1 VM Instance: e2-highmem-2 (2 vCPU, 16GB RAM) - Monitoring (Prometheus, Grafana, SonarQube)
- Total: 2 nodos GKE + 1 VM

**Costos Estimados**:
- e2-medium (nodos GKE): ~$24 USD/nodo/mes
- 2 nodos e2-medium: ~$48 USD/mes
- e2-highmem-2 (VM monitoring): ~$40 USD/mes
- GKE Cluster Management: $0.10/hora (~$73/mes) - **Primer cluster gratis en cuenta de prueba**
- **Total Mensual**: ~$88 USD/mes (con cluster management gratis: ~$88 USD/mes)

**Créditos Disponibles**: $300 USD de crédito de prueba 

**Duración Estimada con Créditos**: ~3-4 meses (dependiendo de uso)

## Desglose Detallado de Costos

### Azure Development

| Recurso | Tipo | Cantidad | Costo Unitario | Costo Mensual |
|---------|------|----------|----------------|---------------|
| Nodo System Pool | Standard_B2s | 1 | ~$15-20/mes | ~$15-20 |
| Nodo Production Pool | Standard_B2s | 1 | ~$15-20/mes | ~$15-20 |
| AKS Management | - | 1 | Gratis | $0 |
| **Total** | | | | **~$30-40/mes** |

### Azure Staging

| Recurso | Tipo | Cantidad | Costo Unitario | Costo Mensual |
|---------|------|----------|----------------|---------------|
| Nodo System Pool | Standard_B2s | 1 | ~$15-20/mes | ~$15-20 |
| Nodo Production Pool | Standard_B2s | 1 | ~$15-20/mes | ~$15-20 |
| AKS Management | - | 1 | Gratis | $0 |
| **Total** | | | | **~$30-40/mes** |

### GCP Production

| Recurso | Tipo | Cantidad | Costo Unitario | Costo Mensual |
|---------|------|----------|----------------|---------------|
| Nodo System Pool | e2-medium | 1 | ~$24/mes | ~$24 |
| Nodo Production Pool | e2-medium | 1 | ~$24/mes | ~$24 |
| VM Monitoring | e2-highmem-2 | 1 | ~$40/mes | ~$40 |
| GKE Cluster Management | - | 1 | Gratis (primer cluster) | $0 |
| **Total** | | | | **~$88/mes** |

### Resumen Total

| Ambiente | Costo Mensual | Créditos Disponibles |
|----------|---------------|---------------------|
| Azure Dev | ~$30-40 | $100 USD (cuenta de estudiante) |
| Azure Stage | ~$30-40 | $100 USD (cuenta de estudiante) |
| GCP Prod | ~$88 | $300 USD (90 días) |
| **TOTAL** | **~$148-168/mes** | **$0 con créditos** |

## Comparación de Especificaciones

### Azure vs GCP

| Característica | Azure (Standard_B2s) | GCP (e2-medium) |
|----------------|---------------------|------------------|
| vCPUs | 2 | 2 |
| RAM | 2 GB | 4 GB |
| Costo/nodo | ~$15-20/mes | ~$24/mes |
| Ventaja | Menor costo | Más RAM (2x) |

**Nota**: GCP proporciona el doble de RAM (4GB vs 2GB) por un costo ligeramente mayor, lo cual es beneficioso para aplicaciones que requieren más memoria.

## Optimización de Costos

### Estrategias Implementadas

1. **Uso de Créditos de Estudiante**: Ambos ambientes de Azure utilizan cuentas de estudiante con créditos gratuitos
2. **Cuenta de Prueba GCP**: Producción utiliza cuenta de prueba con $300 USD de crédito
3. **Tamaños de Máquina Optimizados**: Uso de máquinas pequeñas (Standard_B2s, e2-medium) adecuadas para desarrollo y pruebas
4. **Cluster Único por Ambiente**: Un solo cluster por ambiente reduce costos de gestión

### Estrategias Adicionales Recomendadas

1. **Auto-scaling**: Configurar auto-scaling para aumentar/reducir nodos según demanda
2. **Preemptible/Spot Instances**: Usar instancias preemptibles para cargas de trabajo no críticas (hasta 80% de ahorro)
3. **Apagar Recursos**: Apagar clusters durante períodos de no uso (noches, fines de semana)
4. **Máquinas Más Pequeñas**: Considerar e2-small (2 vCPU, 2GB RAM) para desarrollo (~$13/mes)
5. **Clusters Zonales**: Usar clusters zonales en lugar de regionales para reducir costos de gestión

## Monitoreo de Costos

### Azure

- Portal de Azure: Dashboard de costos y facturación
- Alertas de presupuesto: Configurar alertas cuando se alcance cierto porcentaje de créditos
- Análisis de costos: Revisar desglose por recurso y servicio

### GCP

- Google Cloud Console: Página de facturación
- Budget Alerts: Configurar alertas de presupuesto
- Cost Breakdown: Análisis detallado por proyecto, servicio y recurso

## Factores que Afectan los Costos

1. **Uptime**: Los recursos se facturan por hora de uso, incluso si están inactivos
2. **Tráfico de Red**: Transferencia de datos fuera de la región puede generar costos adicionales
3. **Almacenamiento**: Discos persistentes y snapshots generan costos adicionales
4. **Load Balancers**: IPs públicas estáticas y Load Balancers tienen costos asociados
5. **Escalado**: Aumentar el número de nodos incrementa los costos proporcionalmente

## Proyección de Costos a Largo Plazo

### Con Créditos (Actual)

- **Período de Créditos**: Mientras duren los créditos de estudiante y prueba
- **Costo Real**: $0 USD/mes
- **Duración Estimada**:
  - Azure: ~2-3 meses con $100 USD de crédito por cuenta
  - GCP: ~3-4 meses con $300 USD de crédito

### Sin Créditos (Post-Trial)

- **Costo Mensual Estimado**: ~$148-168 USD/mes
- **Costo Anual Estimado**: ~$1,776-2,016 USD/año

### Recomendaciones Post-Trial

1. **Consolidar Ambientes**: Considerar usar un solo ambiente para dev/stage
2. **Reducir Recursos**: Usar máquinas más pequeñas o menos nodos
3. **Migrar a Cloud Gratuito**: Considerar migrar a tier gratuito de servicios cloud
4. **Auto-scaling Agresivo**: Reducir a mínimo de nodos cuando no hay uso
5. **Preemptible Instances**: Usar instancias preemptibles para desarrollo

## Notas Importantes

### Azure Student Accounts

- Cada cuenta de estudiante incluye $100 USD de crédito

### GCP Trial Account

- $300 USD de crédito 

### Facturación

- Los costos se facturan por hora de uso
- Los recursos se facturan incluso si están detenidos (pero no eliminados)
- Los Load Balancers y IPs públicas tienen costos incluso sin tráfico
- Monitorear regularmente el uso para evitar sorpresas

## Referencias

- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [Azure for Students](https://azure.microsoft.com/free/students/)
- [GCP Free Trial](https://cloud.google.com/free)

