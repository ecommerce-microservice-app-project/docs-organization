# Externalized Configuration Pattern (Patrón de Configuración Externa)

## Tabla de Contenidos

1. [¿Qué es el Externalized Configuration Pattern?](#qué-es-el-externalized-configuration-pattern)
2. [¿Por qué es importante para microservicios?](#por-qué-es-importante-para-microservicios)
3. [Implementación EXISTENTE en este sistema](#implementación-existente-en-este-sistema)
4. [Arquitectura de configuración](#arquitectura-de-configuración)
5. [Cómo funciona](#cómo-funciona)
6. [Mejoras implementadas](#mejoras-implementadas)
7. [Cómo demostrarlo](#cómo-demostrarlo)
8. [Ejemplos prácticos de uso](#ejemplos-prácticos-de-uso)
9. [Ventajas y limitaciones](#ventajas-y-limitaciones)
10. [Mejoras futuras sugeridas](#mejoras-futuras-sugeridas)

---

## ¿Qué es el Externalized Configuration Pattern?

El **Externalized Configuration Pattern** es un patrón arquitectónico que consiste en **separar la configuración de la aplicación del código fuente**, permitiendo que los valores de configuración sean modificados sin necesidad de recompilar o redesplegar la aplicación.

### Principio Fundamental

```
┌──────────────────────────────────────────────────────┐
│   ANTES (Hardcoded Configuration)                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  public class OrderService {                        │
│      private String userServiceUrl =                │
│          "http://localhost:8700";  // ← Hardcoded  │
│      private int maxRetries = 3;   // ← Hardcoded  │
│  }                                                   │
│                                                      │
│   Cambiar requiere: recompilar + redesplegar      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│   DESPUÉS (Externalized Configuration)            │
├──────────────────────────────────────────────────────┤
│                                                      │
│  public class OrderService {                        │
│      @Value("${user.service.url}")                  │
│      private String userServiceUrl;                 │
│                                                      │
│      @Value("${retry.max.attempts}")                │
│      private int maxRetries;                        │
│  }                                                   │
│                                                      │
│   Cambiar requiere: actualizar ConfigMap          │
│   NO requiere recompilar ni redesplegar           │
└──────────────────────────────────────────────────────┘
```

### Concepto Clave

La configuración se almacena en **fuentes externas** (archivos, variables de entorno, servidores de configuración) y se **inyecta** en la aplicación en tiempo de ejecución.

---

## ¿Por qué es importante para microservicios?

### Desafíos de Configuración en Microservicios

En una arquitectura de microservicios con múltiples servicios y ambientes:

```
┌─────────────────────────────────────────────────────────────┐
│  10 Microservicios × 3 Ambientes = 30 Configuraciones       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Services:                    Environments:                 │
│  - order-service              - dev (development)           │
│  - user-service               - stage (staging)             │
│  - product-service            - prod (production)           │
│  - payment-service                                          │
│  - shipping-service           Cada combinación necesita:   │
│  - favourite-service          - URLs diferentes            │
│  - proxy-client               - Credenciales diferentes    │
│  - api-gateway                - Parámetros diferentes      │
│  - service-discovery          - Timeouts diferentes        │
│  - cloud-config                                            │
│                                                             │
│  SIN Externalized Config:                                  │
│   30 archivos WAR/JAR diferentes                         │
│   Recompilar para cada cambio                            │
│   Imposible mantener                                     │
│                                                             │
│  CON Externalized Config:                                  │
│   1 artifact (JAR) para todos los ambientes              │
│   Configuración separada por ambiente                    │
│   Cambios sin recompilar                                 │
└─────────────────────────────────────────────────────────────┘
```

### Beneficios Clave

1. **Mismo artifact en todos los ambientes**: Un JAR, múltiples configs
2. **Cambios operacionales sin deploy**: Ajustar parámetros en caliente
3. **Configuración por ambiente**: dev, stage, prod tienen valores diferentes
4. **Gestión centralizada**: Un solo lugar para toda la configuración
5. **Seguridad mejorada**: Secrets separados del código
6. **Auditoría**: Historial de cambios en la configuración
7. **Feature toggles**: Activar/desactivar funcionalidades sin deploy

---

## Implementación EXISTENTE en este sistema

Este sistema **YA TIENE** implementado el Externalized Configuration Pattern en **3 niveles**:

### Nivel 1: Spring Cloud Config Server (Centralizado)

**Servicio**: `cloud-config-organization`
**Puerto**: 9296
**Repositorio Git**: [https://github.com/SelimHorri/cloud-config-server](https://github.com/SelimHorri/cloud-config-server)

**Descripción**: Servidor centralizado que proporciona configuración a todos los microservicios desde un repositorio Git.

### Nivel 2: ConfigMaps de Kubernetes (Por Ambiente)

**Ubicación**: `kubernetes-organization/k8s/<service>/configmap.yaml`

**Descripción**: Configuración específica de Kubernetes inyectada como variables de entorno en los pods.

### Nivel 3: Perfiles de Spring (application-{env}.yml)

**Archivos**:

- `application-dev.yml` - Configuración de desarrollo
- `application-stage.yml` - Configuración de staging
- `application-prod.yml` - Configuración de producción

**Descripción**: Perfiles de Spring Boot activados según el ambiente.

---

## Arquitectura de configuración

### Vista General del Sistema

```
┌─────────────────────────────────────────────────────────────────┐
│                     FUENTES DE CONFIGURACIÓN                     │
└────┬──────────────────────────┬──────────────────────────┬──────┘
     │                          │                          │
     │                          │                          │
     ↓                          ↓                          ↓
┌──────────────────┐   ┌─────────────────┐   ┌──────────────────────┐
│  Git Repository  │   │  ConfigMaps     │   │  Environment         │
│  (GitHub)        │   │  (Kubernetes)   │   │  Variables           │
│                  │   │                 │   │                      │
│  ├─dev.yml       │   │  order-service  │   │  SPRING_PROFILES_    │
│  ├─stage.yml     │   │    -config      │   │  ACTIVE=prod         │
│  └─prod.yml      │   │                 │   │                      │
└────────┬─────────┘   └────────┬────────┘   └──────────┬───────────┘
         │                      │                        │
         │                      │                        │
         ↓                      ↓                        ↓
┌────────────────────────────────────────────────────────────────────┐
│               SPRING CLOUD CONFIG SERVER                           │
│               (cloud-config-organization:9296)                     │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Agrega y combina configuración de múltiples fuentes     │   │
│  │  - Prioridad: Git > ConfigMap > application.yml          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 │ HTTP /order-service/dev
                                 │
                                 ↓
┌────────────────────────────────────────────────────────────────────┐
│                    MICROSERVICIOS                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ order-service│  │ user-service │  │ product-     │ ...       │
│  │              │  │              │  │  service     │           │
│  │ @Value(...)  │  │ @Value(...)  │  │ @Value(...)  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└────────────────────────────────────────────────────────────────────┘
```

### Flujo de Configuración (Startup)

```
┌────────────────────────────────────────────────────────────────┐
│  PASO 1: Pod de order-service inicia                           │
├────────────────────────────────────────────────────────────────┤
│  - Kubernetes inyecta variables de entorno desde ConfigMap    │
│    SPRING_CONFIG_IMPORT=configserver:http://cloud-config:9296 │
│    SPRING_PROFILES_ACTIVE=prod                                │
│    RETRY_MAX_ATTEMPTS=3                                       │
│    CB_FAILURE_RATE=50                                         │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────────┐
│  PASO 2: Spring Boot lee variables de entorno                 │
├────────────────────────────────────────────────────────────────┤
│  spring:                                                       │
│    application:                                                │
│      name: ORDER-SERVICE                                       │
│    profiles:                                                   │
│      active: ${SPRING_PROFILES_ACTIVE:dev}  → "prod"         │
│    config:                                                     │
│      import: ${SPRING_CONFIG_IMPORT}  → "configserver:..."   │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────────┐
│  PASO 3: Conecta a Config Server                              │
├────────────────────────────────────────────────────────────────┤
│  GET http://cloud-config:9296/order-service/prod              │
│                                                                │
│  Config Server responde con:                                  │
│  {                                                             │
│    "name": "order-service",                                   │
│    "profiles": ["prod"],                                      │
│    "propertySources": [...]                                   │
│  }                                                             │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────────┐
│  PASO 4: Spring Boot construye PropertySource hierarchy       │
├────────────────────────────────────────────────────────────────┤
│  Prioridad (mayor a menor):                                    │
│  1. Variables de entorno (RETRY_MAX_ATTEMPTS=3)              │
│  2. application-prod.yml del Config Server                    │
│  3. application.yml del Config Server                         │
│  4. application-prod.yml local                                │
│  5. application.yml local                                     │
└────────────────────────┬───────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────────┐
│  PASO 5: Inyección de propiedades con @Value                  │
├────────────────────────────────────────────────────────────────┤
│  @Value("${RETRY_MAX_ATTEMPTS:3}")                            │
│  private int retryMaxAttempts;  → 3                           │
│                                                                │
│  @Value("${CB_FAILURE_RATE:50}")                              │
│  private int cbFailureRate;  → 50                             │
└────────────────────────────────────────────────────────────────┘
```

---

## Cómo funciona

### 1. Spring Cloud Config Server

#### Configuración del Config Server

**Archivo**: `cloud-config-organization/src/main/resources/application.yml`

```yaml
server:
  port: 9296

spring:
  application:
    name: CLOUD-CONFIG
  cloud:
    config:
      server:
        git:
          uri: https://github.com/SelimHorri/cloud-config-server
          clone-on-start: true
```

**Qué hace**:

1. Clona el repositorio Git al arrancar
2. Expone endpoint REST: `/{application}/{profile}`
3. Sirve configuración combinando archivos del repo

#### Estructura del Repositorio Git

```
cloud-config-server/
├── application.yml              # Configuración compartida por todos
├── application-dev.yml          # Configuración común de dev
├── application-stage.yml        # Configuración común de stage
├── application-prod.yml         # Configuración común de prod
├── order-service.yml            # Específico de order-service
├── order-service-dev.yml        # order-service en dev
├── order-service-stage.yml      # order-service en stage
└── order-service-prod.yml       # order-service en prod
```

**Ejemplo de request**:

```bash
curl http://cloud-config:9296/order-service/prod
```

**Respuesta** (configuración combinada):

```json
{
  "name": "order-service",
  "profiles": ["prod"],
  "label": null,
  "version": "abc123def",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/.../order-service-prod.yml",
      "source": {
        "server.port": 8300,
        "eureka.client.serviceUrl.defaultZone": "http://eureka-prod:8761/eureka/"
      }
    },
    {
      "name": "https://github.com/.../application-prod.yml",
      "source": {
        "logging.level.root": "INFO"
      }
    }
  ]
}
```

### 2. ConfigMaps de Kubernetes

#### Ejemplo Real: order-service ConfigMap

**Archivo**: `kubernetes-organization/k8s/order-service/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: ecommerce-prod
data:
  # Conexiones a servicios
  EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: "http://service-discovery.ecommerce-prod.svc.cluster.local:8761/eureka/"
  SPRING_CONFIG_IMPORT: "optional:configserver:http://cloud-config.ecommerce-prod.svc.cluster.local:9296"
  SPRING_PROFILES_ACTIVE: "prod"
  SPRING_ZIPKIN_BASE_URL: "http://zipkin.ecommerce-prod.svc.cluster.local:9411/"

  # Eureka configuration
  EUREKA_INSTANCE_PREFER_IP_ADDRESS: "true"
  EUREKA_INSTANCE_HOSTNAME: "order-service.ecommerce-prod.svc.cluster.local"

  # Parámetros de Resiliencia (MEJORA IMPLEMENTADA)
  RETRY_MAX_ATTEMPTS: "3"
  RETRY_DELAY: "1000"
  RETRY_MULTIPLIER: "2"
  CB_FAILURE_RATE: "50"
  CB_WAIT_DURATION: "10s"
  CB_SLIDING_WINDOW: "10"
```

#### Cómo se aplica al Deployment

**Archivo**: `kubernetes-organization/k8s/order-service/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry/order-service:0.1.0
          envFrom:
            - configMapRef:
                name: order-service-config # ← Inyecta TODAS las variables
```

**Resultado**: Todas las key-value del ConfigMap se convierten en variables de entorno dentro del container.

### 3. Uso en el Código

#### Inyección con @Value

```java
@Service
public class CartServiceImpl {

    @Value("${RETRY_MAX_ATTEMPTS:3}")
    private int retryMaxAttempts;

    @Value("${CB_FAILURE_RATE:50}")
    private int cbFailureRate;

    // Spring inyecta automáticamente los valores
}
```

#### Inyección con @ConfigurationProperties

```java
@Configuration
@ConfigurationProperties(prefix = "app.resilience")
public class ResilienceConfig {

    private RetryConfig retry;
    private CircuitBreakerConfig circuitBreaker;

    @Data
    public static class RetryConfig {
        private int maxAttempts = 3;
        private long delay = 1000;
        private int multiplier = 2;
    }

    @Data
    public static class CircuitBreakerConfig {
        private int failureRateThreshold = 50;
        private String waitDuration = "10s";
        private int slidingWindowSize = 10;
    }

    // Getters and Setters
}
```

**Uso**:

```java
@Service
public class OrderService {

    @Autowired
    private ResilienceConfig config;

    public void processOrder() {
        int maxAttempts = config.getRetry().getMaxAttempts();
        // ...
    }
}
```

---

## Mejoras implementadas

### Mejora 1: Externalizar Parámetros de Resiliencia

**Antes** (hardcoded en anotaciones):

```java
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
```

**Después** (configurable externamente):

**application.yml**:

```yaml
app:
  resilience:
    retry:
      max-attempts: ${RETRY_MAX_ATTEMPTS:3}
      delay: ${RETRY_DELAY:1000}
      multiplier: ${RETRY_MULTIPLIER:2}
```

**ConfigMap**:

```yaml
RETRY_MAX_ATTEMPTS: "3" # Cambiar a 5 para más reintentos
RETRY_DELAY: "1000" # Cambiar a 2000 para delays más largos
```

**Beneficio**: Operadores pueden ajustar comportamiento de reintentos sin recompilar.

### Mejora 2: Configuración de Circuit Breaker Externa

**application.yml**:

```yaml
app:
  resilience:
    circuit-breaker:
      failure-rate-threshold: ${CB_FAILURE_RATE:50}
      wait-duration: ${CB_WAIT_DURATION:10s}
      sliding-window-size: ${CB_SLIDING_WINDOW:10}

resilience4j:
  circuitbreaker:
    instances:
      userService:
        failure-rate-threshold: ${app.resilience.circuit-breaker.failure-rate-threshold}
        wait-duration-in-open-state: ${app.resilience.circuit-breaker.wait-duration}
        sliding-window-size: ${app.resilience.circuit-breaker.sliding-window-size}
```

**ConfigMap**:

```yaml
CB_FAILURE_RATE: "50" # Aumentar a 60 para ser más tolerante
CB_WAIT_DURATION: "10s" # Aumentar a 30s para dar más tiempo
CB_SLIDING_WINDOW: "10" # Aumentar a 20 para ventana más grande
```

**Beneficio**: Tuning de resiliencia en producción sin deployments.

### Mejora 3: Separación por Ambiente

#### Desarrollo (dev)

```yaml
# ConfigMap en namespace ecommerce-dev
RETRY_MAX_ATTEMPTS: "3" # Menos reintentos para feedback rápido
CB_WAIT_DURATION: "5s" # Tiempos cortos para testing
SPRING_PROFILES_ACTIVE: "dev"
```

#### Staging (stage)

```yaml
# ConfigMap en namespace ecommerce-stage
RETRY_MAX_ATTEMPTS: "4" # Intermedio
CB_WAIT_DURATION: "10s" # Intermedio
SPRING_PROFILES_ACTIVE: "stage"
```

#### Producción (prod)

```yaml
# ConfigMap en namespace ecommerce-prod
RETRY_MAX_ATTEMPTS: "5" # Más reintentos para máxima resiliencia
CB_WAIT_DURATION: "30s" # Más tiempo para recuperación
SPRING_PROFILES_ACTIVE: "prod"
```

---

## Cómo demostrarlo

### Demo 1: Cambiar configuración sin recompilar

#### Escenario: Aumentar reintentos de 3 a 5

```bash
# Paso 1: Verificar configuración actual
kubectl get configmap order-service-config -n ecommerce-dev -o yaml | grep RETRY_MAX_ATTEMPTS

# Output:
# RETRY_MAX_ATTEMPTS: "3"

# Paso 2: Hacer request y ver logs (3 reintentos)
kubectl scale deployment user-service --replicas=0 -n ecommerce-dev
curl http://api.../order-service/api/carts/1
kubectl logs -f deployment/order-service -n ecommerce-dev

# Logs muestran 3 intentos

# Paso 3: Editar ConfigMap
kubectl edit configmap order-service-config -n ecommerce-dev
# Cambiar RETRY_MAX_ATTEMPTS: "5"

# Paso 4: Reiniciar pod (sin recompilar!)
kubectl rollout restart deployment/order-service -n ecommerce-dev

# Paso 5: Esperar que el nuevo pod esté listo
kubectl rollout status deployment/order-service -n ecommerce-dev

# Paso 6: Hacer request nuevamente
curl http://api.../order-service/api/carts/1
kubectl logs -f deployment/order-service -n ecommerce-dev

# Logs ahora muestran 5 intentos
```

**Tiempo total**: ~2 minutos (vs horas si se requiriera recompilar)

### Demo 2: Configuración diferente por ambiente

```bash
# Ver configuración en DEV
kubectl get configmap order-service-config -n ecommerce-dev -o yaml | grep CB_WAIT_DURATION
# Output: CB_WAIT_DURATION: "5s"

# Ver configuración en STAGE
kubectl get configmap order-service-config -n ecommerce-stage -o yaml | grep CB_WAIT_DURATION
# Output: CB_WAIT_DURATION: "10s"

# Ver configuración en PROD
kubectl get configmap order-service-config -n ecommerce-prod -o yaml | grep CB_WAIT_DURATION
# Output: CB_WAIT_DURATION: "30s"

# Mismo JAR, diferentes configuraciones
```

### Demo 3: Actualización en vivo con Spring Cloud Config

```bash
# Paso 1: Clonar el repo de configuración
git clone https://github.com/SelimHorri/cloud-config-server.git
cd cloud-config-server

# Paso 2: Modificar configuración
echo "new.property: nueva-caracteristica" >> order-service-prod.yml

# Paso 3: Commit y push
git add order-service-prod.yml
git commit -m "Add new property"
git push origin main

# Paso 4: Refrescar configuración SIN reiniciar pod
curl -X POST http://order-service:8080/order-service/actuator/refresh

# Paso 5: Verificar nueva propiedad
curl http://order-service:8080/order-service/actuator/env | jq '.propertySources[] | select(.name | contains("cloud-config"))'

# Nueva propiedad disponible
```

---

## Ejemplos prácticos de uso

### Caso 1: Feature Toggle (Activar/Desactivar funcionalidad)

**ConfigMap**:

```yaml
FEATURE_NEW_CHECKOUT: "false" # Desactivado en prod
```

**Código**:

```java
@Service
public class OrderService {

    @Value("${FEATURE_NEW_CHECKOUT:false}")
    private boolean newCheckoutEnabled;

    public OrderDto checkout(CartDto cart) {
        if (newCheckoutEnabled) {
            return newCheckoutFlow(cart);  // Nueva implementación
        } else {
            return legacyCheckoutFlow(cart);  // Implementación anterior
        }
    }
}
```

**Ventaja**: Activar nueva funcionalidad en producción sin deploy, solo cambiando el ConfigMap.

### Caso 2: Ajuste de Timeouts en Producción

**Problema**: Timeouts muy cortos causan fallos en producción.

**Solución**:

```bash
# Sin recompilar, aumentar timeouts
kubectl edit configmap order-service-config -n ecommerce-prod

# Cambiar:
# HTTP_CLIENT_TIMEOUT: "5000"  → "10000"
# HTTP_READ_TIMEOUT: "5000"    → "15000"

kubectl rollout restart deployment/order-service -n ecommerce-prod
```

**Resultado**: Problema resuelto en 2 minutos.

### Caso 3: Configuración de Rate Limiting

**ConfigMap**:

```yaml
RATE_LIMIT_PER_MINUTE: "100" # 100 requests/minuto
RATE_LIMIT_BURST: "20" # Burst de 20 requests
```

**Código**:

```java
@Configuration
public class RateLimitConfig {

    @Value("${RATE_LIMIT_PER_MINUTE:100}")
    private int requestsPerMinute;

    @Value("${RATE_LIMIT_BURST:20}")
    private int burstSize;

    @Bean
    public RateLimiter rateLimiter() {
        return RateLimiter.of("api", RateLimiterConfig.custom()
            .limitForPeriod(requestsPerMinute)
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .build());
    }
}
```

**Ventaja**: Ajustar rate limiting según carga sin deploy.

---

## Ventajas y limitaciones

### Ventajas

1. **Un artifact para todos los ambientes**: Mismo JAR en dev/stage/prod
2. **Cambios sin recompilar**: Ajustes operacionales rápidos
3. **Gestión centralizada**: Config Server como fuente única de verdad
4. **Auditoría**: Git history muestra quién cambió qué y cuándo
5. **Feature toggles**: Activar/desactivar funcionalidades dinámicamente
6. **Secrets separados**: Credenciales fuera del código
7. **Configuración tipo**: Type-safe con @ConfigurationProperties
8. **Refresh en caliente**: Spring Cloud Config refresh sin reiniciar

### Limitaciones

1. **Dependencia del Config Server**: Si cae, servicios no pueden arrancar
2. **Complejidad adicional**: Más componentes que mantener
3. **Latencia de startup**: Servicios tardan más en arrancar
4. **Requiere reinicio** (ConfigMap): Cambios en K8s ConfigMap requieren restart
5. **Versionado complejo**: Múltiples fuentes de configuración
6. **Secrets en plaintext**: ConfigMaps no encriptan valores sensibles

### Anti-patrones a evitar

1. **Hardcodear valores**: Contradice el propósito del patrón
2. **No usar valores por defecto**: Siempre proporcionar defaults
3. **Configuración excesiva**: No externalizar TODO
4. **Secrets en ConfigMaps**: Usar Kubernetes Secrets en su lugar
5. **Sin documentación**: Nadie sabe qué configurar
6. **No validar configuración**: Permitir valores inválidos

---

## Mejoras futuras sugeridas

### 1. Kubernetes Secrets para Credenciales

**Problema actual**: Passwords y API keys en ConfigMaps (plaintext)

**Solución**:

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
data:
  database-password: cGFzc3dvcmQxMjM= # base64
  api-key: YWJjZGVmZ2hpams= # base64
```

**Uso en Deployment**:

```yaml
containers:
  - name: order-service
    envFrom:
      - configMapRef:
          name: order-service-config
      - secretRef:
          name: order-service-secrets # ← Agregar Secrets
```

### 2. HashiCorp Vault para Gestión de Secrets

**Arquitectura propuesta**:

```
┌────────────────┐
│  HashiCorp     │
│  Vault         │
│  (Secrets)     │
└────────┬───────┘
         │
         │ API
         ↓
┌────────────────────────────┐
│  Spring Cloud Vault        │
│  (en cada servicio)        │
└────────┬───────────────────┘
         │
         ↓
┌────────────────────────────┐
│  Microservicio             │
│  @Value("${vault.secret}") │
└────────────────────────────┘
```

**Beneficios**:

- Encriptación de secrets
- Rotación automática de credenciales
- Auditoría de acceso
- Dynamic secrets (credenciales temporales)

### 3. ConfigMaps Dinámicos (sin restart)

**Problema**: Cambios en ConfigMap requieren reinicio de pod

**Solución**: Usar ConfigMap Reloader

```yaml
# Instalar Reloader
kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml

# Anotar Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  annotations:
    configmap.reloader.stakater.com/reload: "order-service-config"
```

**Resultado**: Cambios en ConfigMap reinician automáticamente el pod.

### 4. Spring Cloud Config con Refresh Automático

**Implementar**:

```java
@RefreshScope  // ← Permite refresh en caliente
@Service
public class CartServiceImpl {

    @Value("${retry.max.attempts}")
    private int retryMaxAttempts;  // Se actualiza al hacer refresh
}
```

**Trigger refresh**:

```bash
# Cambiar config en Git
# Luego:
curl -X POST http://order-service:8080/order-service/actuator/refresh

# retryMaxAttempts se actualiza SIN reiniciar el pod
```

### 5. Configuración A/B Testing

**ConfigMap**:

```yaml
FEATURE_NEW_ALGO_PERCENTAGE: "20" # 20% de usuarios ven nueva versión
```

**Código**:

```java
@Service
public class RecommendationService {

    @Value("${FEATURE_NEW_ALGO_PERCENTAGE:0}")
    private int newAlgoPercentage;

    public List<Product> getRecommendations(User user) {
        int random = ThreadLocalRandom.current().nextInt(100);
        if (random < newAlgoPercentage) {
            return newAlgorithm(user);  // 20% de usuarios
        } else {
            return legacyAlgorithm(user);  // 80% de usuarios
        }
    }
}
```

**Rollout gradual**: Aumentar porcentaje de 20% → 50% → 100% sin deployments.

### 6. Observabilidad de Configuración

**Implementar**:

```bash
# Endpoint que muestra configuración actual
GET /actuator/configprops

# Endpoint que muestra de dónde viene cada propiedad
GET /actuator/env

# Logs de cambios de configuración
2025-01-26 10:30:00 INFO Configuration changed: RETRY_MAX_ATTEMPTS 3 → 5
```

### 7. Validación de Configuración

**Implementar**:

```java
@Configuration
@Validated
public class ResilienceConfig {

    @Min(1)
    @Max(10)
    private int retryMaxAttempts = 3;

    @Min(100)
    @Max(60000)
    private long retryDelay = 1000;

    // Spring validará al arrancar, fallará si valores inválidos
}
```

### 8. Configuración Multi-Tenant

**Para el futuro** (si se agregan múltiples clientes):

```
Config Server
  ├─ tenant-acme/
  │    ├─ order-service-prod.yml
  │    └─ user-service-prod.yml
  └─ tenant-globex/
       ├─ order-service-prod.yml
       └─ user-service-prod.yml
```

---

## Conclusión

El **Externalized Configuration Pattern** es fundamental para operar microservicios en múltiples ambientes de manera eficiente. Este sistema **ya implementa** este patrón de manera robusta con:

1.  **Spring Cloud Config Server**: Configuración centralizada desde Git
2.  **Kubernetes ConfigMaps**: Configuración específica de K8s
3.  **Perfiles de Spring**: Configuración por ambiente (dev/stage/prod)
4.  **Externalización de parámetros de resiliencia** (MEJORA): Retry y Circuit Breaker configurables

### Impacto de las Mejoras Implementadas

**Antes**:

```
Cambiar max_retries de 3 a 5:
1. Modificar anotación @Retryable en código
2. Recompilar (mvn clean package)
3. Construir nueva imagen Docker
4. Push a registry
5. Actualizar deployment.yaml
6. Aplicar con kubectl apply
Tiempo: ~30-60 minutos
```

**Después** (con Externalized Config):

```
Cambiar max_retries de 3 a 5:
1. kubectl edit configmap order-service-config
2. kubectl rollout restart deployment/order-service
Tiempo: ~2 minutos
```

**Mejora**: **15-30x más rápido** ⚡

### Próximos Pasos Recomendados

1. Implementar **Kubernetes Secrets** para credenciales
2. Evaluar **HashiCorp Vault** para gestión avanzada de secrets
3. Agregar **ConfigMap Reloader** para actualizaciones sin downtime
4. Habilitar **@RefreshScope** para refresh en caliente
5. Implementar dashboards de **configuración activa** en Grafana

---

**Estado actual**: IMPLEMENTADO Y FUNCIONANDO
**Mejoras**: PARÁMETROS DE RESILIENCIA EXTERNALIZADOS
**Framework**: Spring Cloud Config + Kubernetes ConfigMaps
**Última actualización**: Enero 2025
