# Retry Pattern (Patrón de Reintentos)

## Tabla de Contenidos

1. [¿Qué es el Retry Pattern?](#qué-es-el-retry-pattern)
2. [¿Por qué es importante para microservicios?](#por-qué-es-importante-para-microservicios)
3. [Cuándo usar este patrón](#cuándo-usar-este-patrón)
4. [Implementación en este sistema](#implementación-en-este-sistema)
5. [Cómo funciona](#cómo-funciona)
6. [Parámetros configurables](#parámetros-configurables)
7. [Cómo demostrarlo](#cómo-demostrarlo)
8. [Logs esperados](#logs-esperados)
9. [Ventajas y limitaciones](#ventajas-y-limitaciones)
10. [Mejores prácticas](#mejores-prácticas)

---

## ¿Qué es el Retry Pattern?

El **Retry Pattern** es un patrón de resiliencia que automáticamente **reintenta operaciones que han fallado**, asumiendo que el fallo es temporal (transient failure). En lugar de fallar inmediatamente cuando una operación no tiene éxito, el sistema espera un tiempo determinado y vuelve a intentar la operación.

### Concepto Clave

```
Request → Fallo (timeout, red inestable)
    ↓
Esperar 1 segundo
    ↓
Reintento #1 → Fallo
    ↓
Esperar 2 segundos (exponential backoff)
    ↓
Reintento #2 → Fallo
    ↓
Esperar 4 segundos
    ↓
Reintento #3 → ✅ Éxito o ❌ Fallo definitivo
```

---

## ¿Por qué es importante para microservicios?

En arquitecturas de microservicios, los servicios se comunican constantemente a través de la red. Esta comunicación está sujeta a **fallos temporales** que pueden ocurrir por diversas razones:

### Problemas Comunes en Microservicios:

1. **Latencia de red temporal**: Paquetes perdidos, congestión momentánea
2. **Servicio sobrecargado**: El servicio destino está procesando muchas solicitudes
3. **Inicialización lenta**: El servicio acaba de reiniciarse y está cargando recursos
4. **Actualizaciones del servicio**: Deploy en progreso, rolling updates
5. **Problemas de DNS**: Resolución temporal de nombres

Sin el Retry Pattern, estos fallos temporales causarían que las solicitudes fallen innecesariamente, degradando la experiencia del usuario.

### Beneficios:

- ✅ **Resiliencia**: Recuperación automática de fallos temporales
- ✅ **Mejor experiencia de usuario**: Operaciones exitosas a pesar de problemas transitorios
- ✅ **Reducción de falsos errores**: Evita reportar errores que se resolverían con un reintento
- ✅ **Sin intervención manual**: Automático y transparente

---

## Cuándo usar este patrón

### ✅ USAR cuando:

- Llamadas a servicios HTTP/REST externos
- Operaciones que pueden fallar temporalmente
- Fallos de red transitorios
- Servicios que ocasionalmente devuelven timeouts
- Bases de datos que pueden tener bloqueos momentáneos

### ❌ NO USAR cuando:

- Errores de lógica de negocio (400 Bad Request, 422 Unprocessable Entity)
- Errores de autenticación (401, 403)
- Recursos no encontrados (404)
- Operaciones no idempotentes sin garantías
- Errores permanentes del servidor

---

## Implementación en este sistema

### Arquitectura del Sistema

```
┌─────────────────────┐
│  Order Service      │
│  (con Retry)        │
└──────────┬──────────┘
           │
           │ HTTP GET /users/{id}
           ↓
┌─────────────────────┐
│  User Service       │
│                     │
└─────────────────────┘
```

### Ubicación del Código

### Código Implementado

#### 1. Dependencias Maven (`pom.xml`)

```xml
<!-- Spring Retry para Retry Pattern -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### 2. Habilitar Retry (`OrderServiceApplication.java`)

```java
@SpringBootApplication
@EnableEurekaClient
@EnableJpaAuditing
@EnableRetry  // ← Habilita Spring Retry
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

#### 3. Aplicar @Retryable al método (`CartServiceImpl.java`)

```java
@Override
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackFindById")
@Retryable(
    value = {RestClientException.class, Exception.class},  // Excepciones a reintentar
    maxAttempts = 3,                                        // Número total de intentos
    backoff = @Backoff(delay = 1000, multiplier = 2)      // Backoff exponencial
)
public CartDto findById(final Integer cartId) {
    log.info("*** CartDto, service; fetch cart by id *");
    return this.cartRepository.findById(cartId)
            .map(CartMappingHelper::map)
            .map(c -> {
                if (c.getUserId() != null && c.getUserDto() != null
                    && c.getUserDto().getUserId() != null) {
                    try {
                        log.info("Attempting to call USER-SERVICE for user ID: {}",
                                c.getUserDto().getUserId());

                        // Llamada a USER-SERVICE (puede fallar)
                        UserDto userDto = this.restTemplate.getForObject(
                            AppConstant.DiscoveredDomainsApi.USER_SERVICE_API_URL
                                + "/" + c.getUserDto().getUserId(),
                            UserDto.class
                        );

                        if (userDto != null) {
                            c.setUserDto(userDto);
                            log.info("Successfully retrieved user data from USER-SERVICE");
                        }
                    } catch (Exception e) {
                        log.error("Failed to call USER-SERVICE for user ID {}: {}",
                                 c.getUserDto().getUserId(), e.getMessage());
                        // Relanzar para que Retry lo capture
                        throw e;
                    }
                }
                return c;
            })
            .orElseThrow(() -> new CartNotFoundException(
                String.format("Cart with id: %d not found", cartId)));
}
```

#### 4. Método @Recover (Fallback cuando todos los reintentos fallan)

```java
@Recover
public CartDto recoverFromUserServiceFailure(RestClientException e, Integer cartId) {
    log.error("All retry attempts exhausted for cart {}. Returning cart without user details.",
              cartId);
    // Devolver el cart sin datos de usuario completos
    return this.cartRepository.findById(cartId)
            .map(CartMappingHelper::map)
            .orElseThrow(() -> new CartNotFoundException(
                String.format("Cart with id: %d not found", cartId)));
}

@Recover
public CartDto recoverFromUserServiceFailure(Exception e, Integer cartId) {
    log.error("All retry attempts exhausted for cart {} due to: {}. " +
              "Returning cart without user details.", cartId, e.getMessage());
    return this.cartRepository.findById(cartId)
            .map(CartMappingHelper::map)
            .orElseThrow(() -> new CartNotFoundException(
                String.format("Cart with id: %d not found", cartId)));
}
```

---

## Cómo funciona

### Flujo de Ejecución Detallado

```
┌────────────────────────────────────────────────────────────┐
│ 1. Cliente solicita Cart #1                               │
│    GET /order-service/api/carts/1                         │
└─────────────────────┬──────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. CartServiceImpl.findById(1) es invocado                 │
│    - Spring detecta @Retryable                             │
│    - Crea un proxy que intercepta el método                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. INTENTO #1 (inmediato)                                  │
│    - Busca Cart en DB → OK                                 │
│    - Llama a USER-SERVICE → ❌ FALLO (Connection Timeout)  │
│    - Excepción: RestClientException                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Spring Retry intercepta la excepción                    │
│    - Verifica: ¿Es RestClientException? → SÍ              │
│    - Verifica: ¿Quedan intentos? → SÍ (2 de 3)           │
│    - ESPERA: delay * multiplier^0 = 1000ms * 1 = 1s      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓ (después de 1 segundo)
┌─────────────────────────────────────────────────────────────┐
│ 5. INTENTO #2 (después de 1s)                              │
│    - Busca Cart en DB → OK                                 │
│    - Llama a USER-SERVICE → ❌ FALLO (Connection Timeout)  │
│    - Excepción: RestClientException                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. Spring Retry intercepta nuevamente                      │
│    - Verifica: ¿Quedan intentos? → SÍ (1 de 3)           │
│    - ESPERA: delay * multiplier^1 = 1000ms * 2 = 2s      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓ (después de 2 segundos)
┌─────────────────────────────────────────────────────────────┐
│ 7. INTENTO #3 (después de 2s adicionales)                  │
│    - Busca Cart en DB → OK                                 │
│    - Llama a USER-SERVICE → ✅ ÉXITO                       │
│    - UserDto recibido correctamente                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Respuesta exitosa al cliente                            │
│    - CartDto con UserDto completo                          │
│    - Total de tiempo: ~3 segundos (1s + 2s + tiempo exec) │
└─────────────────────────────────────────────────────────────┘
```

### Caso: Todos los reintentos fallan

```
Intento #1 → FALLO → Espera 1s
Intento #2 → FALLO → Espera 2s
Intento #3 → FALLO → No más intentos
    ↓
Método @Recover es invocado
    ↓
Devuelve Cart sin UserDto (graceful degradation)
```

---

## Parámetros configurables

### Parámetros en el código (`@Retryable`)

| Parámetro            | Valor                                          | Descripción                                              |
| -------------------- | ---------------------------------------------- | -------------------------------------------------------- |
| `value`              | `{RestClientException.class, Exception.class}` | Excepciones que activan el retry                         |
| `maxAttempts`        | `3`                                            | Número total de intentos (1 original + 2 reintentos)     |
| `backoff.delay`      | `1000`                                         | Delay inicial en milisegundos antes del primer reintento |
| `backoff.multiplier` | `2`                                            | Multiplicador para backoff exponencial                   |

### Cálculo de Delays (Exponential Backoff)

```
Intento 1: Inmediato (0ms)
Intento 2: delay * multiplier^0 = 1000ms * 1 = 1 segundo
Intento 3: delay * multiplier^1 = 1000ms * 2 = 2 segundos
Intento 4: delay * multiplier^2 = 1000ms * 4 = 4 segundos (si maxAttempts fuera 4)
```

### Configuración Externa (ConfigMap de Kubernetes)

Los parámetros pueden ajustarse sin recompilar mediante variables de entorno:

**Archivo**: `kubernetes-organization/k8s/order-service/configmap.yaml`

```yaml
# Parámetros de Resiliencia - Retry Pattern
RETRY_MAX_ATTEMPTS: "3" # Cambiar a "5" para más reintentos
RETRY_DELAY: "1000" # Cambiar a "2000" para delays más largos
RETRY_MULTIPLIER: "2" # Cambiar a "3" para crecimiento más rápido
```

**Uso en `application.yml`**:

```yaml
app:
  resilience:
    retry:
      max-attempts: ${RETRY_MAX_ATTEMPTS:3}
      delay: ${RETRY_DELAY:1000}
      multiplier: ${RETRY_MULTIPLIER:2}
```

---

## Cómo demostrarlo

### Prerequisitos

- Cluster de Kubernetes funcionando
- Order Service desplegado
- User Service desplegado
- Acceso a kubectl

### Test 1: Ver reintentos en logs

```bash
# 1. Obtener logs en tiempo real
kubectl logs -f deployment/order-service -n ecommerce-dev

# 2. En otra terminal, hacer request
curl http://api.alianzadelamagiaeterna.com/order-service/api/carts/1

# 3. Observar los logs (verás los intentos)
```

### Test 2: Simular fallo de USER-SERVICE

```bash
# Paso 1: Apagar user-service
kubectl scale deployment user-service --replicas=0 -n ecommerce-dev

# Paso 2: Verificar que está apagado
kubectl get pods -n ecommerce-dev | grep user-service

# Paso 3: Hacer request a order-service
curl -v http://api.alianzadelamagiaeterna.com/order-service/api/carts/1

# Paso 4: Ver logs de order-service (verás 3 intentos)
kubectl logs -f deployment/order-service -n ecommerce-dev

# Paso 5: Levantar user-service de nuevo
kubectl scale deployment user-service --replicas=1 -n ecommerce-dev
```

### Test 3: Recuperación automática (MUY VISUAL)

Este test demuestra la resiliencia del patrón:

```bash
# Paso 1: Apagar user-service
kubectl scale deployment user-service --replicas=0 -n ecommerce-dev

# Paso 2: Hacer request (se quedará esperando)
curl http://api.alianzadelamagiaeterna.com/order-service/api/carts/1 &

# Paso 3: MIENTRAS los reintentos están ocurriendo, levantar user-service
# (hacerlo entre 1-2 segundos después del request)
kubectl scale deployment user-service --replicas=1 -n ecommerce-dev

# Paso 4: Ver cómo el 2do o 3er reintento tiene ÉXITO automáticamente
# El request original se completa exitosamente
```

**Resultado esperado**: El request se completa con éxito a pesar de que user-service estaba caído inicialmente. Esto demuestra recuperación automática.

### Test 4: Cambiar configuración sin recompilar

```bash
# Paso 1: Ver configuración actual
kubectl get configmap order-service-config -n ecommerce-dev -o yaml

# Paso 2: Editar ConfigMap para aumentar reintentos a 5
kubectl edit configmap order-service-config -n ecommerce-dev
# Cambiar RETRY_MAX_ATTEMPTS de "3" a "5"

# Paso 3: Reiniciar pod para aplicar cambios
kubectl rollout restart deployment/order-service -n ecommerce-dev

# Paso 4: Esperar a que el pod esté listo
kubectl rollout status deployment/order-service -n ecommerce-dev

# Paso 5: Repetir Test 2 y verificar que ahora hace 5 intentos
kubectl logs -f deployment/order-service -n ecommerce-dev
```

---

## Logs esperados

### Logs exitosos (con reintentos)

```
2025-01-26 10:15:30 INFO  CartDto, service; fetch cart by id
2025-01-26 10:15:30 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:15:31 ERROR Failed to call USER-SERVICE for user ID 42: Connection timeout
2025-01-26 10:15:32 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:15:33 ERROR Failed to call USER-SERVICE for user ID 42: Connection timeout
2025-01-26 10:15:35 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:15:35 INFO  Successfully retrieved user data from USER-SERVICE
```

**Análisis**:

- **10:15:30**: Intento #1 → Fallo
- **10:15:32**: Intento #2 (después de 1s) → Fallo
- **10:15:35**: Intento #3 (después de 2s adicionales) → Éxito ✅

### Logs cuando todos los reintentos fallan

```
2025-01-26 10:20:00 INFO  CartDto, service; fetch cart by id
2025-01-26 10:20:00 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:20:01 ERROR Failed to call USER-SERVICE for user ID 42: Connection refused
2025-01-26 10:20:02 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:20:03 ERROR Failed to call USER-SERVICE for user ID 42: Connection refused
2025-01-26 10:20:05 INFO  Attempting to call USER-SERVICE for user ID: 42
2025-01-26 10:20:06 ERROR Failed to call USER-SERVICE for user ID 42: Connection refused
2025-01-26 10:20:06 ERROR All retry attempts exhausted for cart 1. Returning cart without user details.
```

**Análisis**:

- Los 3 intentos fallaron
- El método `@Recover` fue invocado
- Se devuelve Cart sin UserDto (degradación elegante)

---

## Ventajas y limitaciones

### ✅ Ventajas

1. **Resiliencia automática**: Maneja fallos temporales sin intervención
2. **Mejora experiencia de usuario**: Reduce errores percibidos
3. **Simple de implementar**: Una anotación y configuración
4. **Transparente**: No afecta la lógica de negocio
5. **Configurable externamente**: Ajustes sin recompilar
6. **Graceful degradation**: `@Recover` permite respuestas parciales
7. **Backoff exponencial**: Reduce carga en servicios sobrecargados

### ⚠️ Limitaciones

1. **Latencia aumentada**: Los reintentos agregan tiempo de respuesta
2. **No para todos los errores**: Solo fallos transitorios
3. **Carga adicional**: Múltiples intentos consumen recursos
4. **Complejidad en debugging**: Los logs muestran múltiples intentos
5. **Requiere idempotencia**: Las operaciones deben ser seguras de repetir
6. **No reemplaza Circuit Breaker**: Ambos deben usarse juntos

### 🚫 Anti-patrones a evitar

1. **Reintentos infinitos**: Siempre establecer `maxAttempts`
2. **Delays muy cortos**: Pueden agravar la carga en servicios caídos
3. **Reintentar errores 4xx**: Errores de cliente no son transitorios
4. **Sin logging**: Imposibilita debugging
5. **Operaciones no idempotentes**: Pueden causar duplicados

---

## Mejores prácticas

### 1. Usar Exponential Backoff

```java
@Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
```

Evita sobrecargar servicios que se están recuperando.

### 2. Limitar número de reintentos

```java
maxAttempts = 3  // ← 3-5 intentos es razonable
```

Demasiados reintentos aumentan latencia sin beneficio.

### 3. Reintentar solo excepciones apropiadas

```java
@Retryable(
    value = {RestClientException.class, SocketTimeoutException.class},
    exclude = {HttpClientErrorException.class}  // No reintentar 4xx
)
```

### 4. Implementar @Recover

Siempre proporcionar fallback para cuando todos los reintentos fallen:

```java
@Recover
public CartDto handleFailure(Exception e, Integer cartId) {
    // Lógica de degradación elegante
}
```

### 5. Configurar timeouts

```java
RestTemplate restTemplate = new RestTemplateBuilder()
    .setConnectTimeout(Duration.ofSeconds(2))
    .setReadTimeout(Duration.ofSeconds(5))
    .build();
```

Sin timeouts, cada intento podría tardar mucho.

### 6. Monitorear métricas

- Tasa de reintentos exitosos vs fallidos
- Latencia promedio con reintentos
- Número de veces que se llega a `@Recover`

### 7. Combinar con Circuit Breaker

```java
@CircuitBreaker(name = "userService")  // ← Primero
@Retryable(maxAttempts = 3)            // ← Segundo
public CartDto findById(Integer id) { }
```

Circuit Breaker evita reintentos cuando el servicio está completamente caído.

### 8. Considerar jitter

Agregar aleatoriedad a los delays para evitar "thundering herd":

```java
@Backoff(delay = 1000, multiplier = 2, random = true)
```

---

## Conclusión

El **Retry Pattern** es fundamental para construir microservicios resilientes que pueden manejar la naturaleza impredecible de las redes distribuidas. Cuando se implementa correctamente con exponential backoff, límites razonables y degradación elegante, proporciona una experiencia de usuario significativamente mejor sin agregar complejidad excesiva al código.

**Integración con otros patrones**:

- Combina perfectamente con **Circuit Breaker** para resiliencia completa
- Soportado por **Externalized Configuration** para ajustes operacionales
- Complementa **Timeout Pattern** para evitar esperas infinitas

---

**Implementado en**: Order Service
**Versión**: 0.1.0
**Framework**: Spring Retry
**Última actualización**: Enero 2025
