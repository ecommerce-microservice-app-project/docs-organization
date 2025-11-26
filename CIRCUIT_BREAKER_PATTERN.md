# Circuit Breaker Pattern (Patrón de Interruptor de Circuito)

## Tabla de Contenidos

1. [¿Qué es el Circuit Breaker Pattern?](#qué-es-el-circuit-breaker-pattern)
2. [Estados del Circuit Breaker](#estados-del-circuit-breaker)
3. [¿Por qué es importante para microservicios?](#por-qué-es-importante-para-microservicios)
4. [Cuándo usar este patrón](#cuándo-usar-este-patrón)
5. [Implementación en este sistema](#implementación-en-este-sistema)
6. [Cómo funciona](#cómo-funciona)
7. [Parámetros configurables](#parámetros-configurables)
8. [Cómo demostrarlo](#cómo-demostrarlo)
9. [Integración con Retry Pattern](#integración-con-retry-pattern)
10. [Métricas y monitoreo](#métricas-y-monitoreo)
11. [Ventajas y limitaciones](#ventajas-y-limitaciones)
12. [Mejores prácticas](#mejores-prácticas)

---

## ¿Qué es el Circuit Breaker Pattern?

El **Circuit Breaker Pattern** es un patrón de resiliencia inspirado en los interruptores eléctricos. Su función es **prevenir que una aplicación intente ejecutar una operación que probablemente fallará**, permitiendo que continue sin esperar el fallo o malgastar recursos.

### Analogía con Interruptor Eléctrico

Imagina el interruptor eléctrico de tu casa:

- **Normal (CLOSED)**: La electricidad fluye normalmente
- **Sobrecarga detectada**: El interruptor se abre automáticamente
- **Abierto (OPEN)**: No permite paso de electricidad, protegiendo el sistema
- **Prueba (HALF_OPEN)**: Después de un tiempo, permite probar si el problema se resolvió

El Circuit Breaker en software funciona exactamente igual con las llamadas a servicios.

---

## Estados del Circuit Breaker

El Circuit Breaker opera en **3 estados** distintos:

### 📊 Diagrama de Estados

```
                    ┌─────────────────────────┐
                    │                         │
                    │   CLOSED (Cerrado)      │
                    │   Estado Normal         │
                    │                         │
                    └────────┬────────────────┘
                             │
                             │ failure_rate_threshold
                             │ alcanzado (ej: 50%)
                             │
                             ↓
                    ┌─────────────────────────┐
                    │                         │
         ┌─────────►│   OPEN (Abierto)        │◄──────────┐
         │          │   Rechaza llamadas      │           │
         │          │                         │           │
         │          └────────┬────────────────┘           │
         │                   │                            │
         │                   │ wait_duration              │
         │ Todas las         │ transcurrido               │ Llamadas
         │ llamadas          │ (ej: 10s)                  │ fallan
         │ fallan            │                            │
         │                   ↓                            │
         │          ┌─────────────────────────┐           │
         │          │                         │           │
         └──────────┤   HALF_OPEN             ├───────────┘
                    │   (Medio Abierto)       │
                    │   Probando...           │
                    │                         │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              │              │              │
    Llamadas exitosas   Llamadas fallan  │
              │              │              │
              ↓              ↓              │
         CLOSED          OPEN              │
```

### 1. 🟢 CLOSED (Cerrado) - Estado Normal

**Descripción**: El circuito está cerrado, las llamadas fluyen normalmente.

**Comportamiento**:

- ✅ Todas las solicitudes se envían al servicio destino
- 📊 Se registran éxitos y fallos en una ventana deslizante (sliding window)
- 🔢 Se calcula continuamente la tasa de fallos (failure rate)
- 🚨 Si la tasa de fallos supera el threshold → Transición a OPEN

**Ejemplo**:

```
Ventana: [✅ ✅ ✅ ❌ ✅ ✅ ✅ ❌ ✅ ✅]
Tasa de fallos: 20% (2 de 10)
Threshold: 50%
Estado: CLOSED (20% < 50%, todo normal)
```

### 2. 🔴 OPEN (Abierto) - Circuito Abierto

**Descripción**: El circuito se ha abierto, el servicio destino se considera no disponible.

**Comportamiento**:

- ❌ **TODAS las solicitudes son rechazadas INMEDIATAMENTE**
- ⚡ No se envían requests al servicio destino (fail-fast)
- 🔀 Se ejecuta automáticamente el método fallback
- ⏱️ Después de `wait_duration` → Transición a HALF_OPEN

**Ventajas**:

- Protege el servicio caído de más carga
- Respuesta inmediata (no hay timeouts)
- Permite que el servicio se recupere

**Ejemplo**:

```
Estado: OPEN
Request llegando → ❌ Rechazado inmediatamente
Fallback: Devolver datos en caché o respuesta degradada
Tiempo en OPEN: 10 segundos
```

### 3. 🟡 HALF_OPEN (Medio Abierto) - Estado de Prueba

**Descripción**: El circuito está probando si el servicio destino se recuperó.

**Comportamiento**:

- 🧪 Permite un número limitado de llamadas de prueba
- 📊 Evalúa si estas llamadas tienen éxito
- ✅ Si las pruebas tienen éxito → Transición a CLOSED
- ❌ Si las pruebas fallan → Transición a OPEN

**Ejemplo**:

```
Estado: HALF_OPEN
Llamadas permitidas: 3
Resultados: [✅ ✅ ✅]
Todas exitosas → Transición a CLOSED ✅

O si:
Resultados: [✅ ❌ ❌]
Fallan → Transición a OPEN ❌ (esperar otros 10s)
```

---

## ¿Por qué es importante para microservicios?

### Problema sin Circuit Breaker

Cuando un servicio falla en una arquitectura de microservicios:

```
Order Service → User Service (CAÍDO)
     ↓
Espera 5s (timeout)
     ↓
Reintenta 3 veces
     ↓
Cada reintento espera 5s
     ↓
Total: 15 segundos de espera
     ↓
100 requests simultáneos = 100 threads bloqueados
     ↓
💥 Order Service se queda sin recursos
💥 Falla en cascada a otros servicios
```

### Solución con Circuit Breaker

```
Order Service → User Service (CAÍDO)
     ↓
Primera llamada: Timeout 5s ❌
Segunda llamada: Timeout 5s ❌
...
Décima llamada: Timeout 5s ❌
     ↓
Circuit Breaker se ABRE (50% de fallos detectados)
     ↓
Llamadas subsecuentes:
     ↓
Rechazadas instantáneamente ⚡
Fallback ejecutado inmediatamente
     ↓
✅ Order Service permanece saludable
✅ User Service tiene tiempo de recuperarse
✅ No hay threads bloqueados
```

### Beneficios Clave

1. **Previene fallos en cascada**: Un servicio caído no derriba todo el sistema
2. **Fail-fast**: Respuestas inmediatas en lugar de timeouts largos
3. **Recuperación automática**: Prueba periódicamente si el servicio se recuperó
4. **Protección del servicio destino**: No lo bombardea cuando está caído
5. **Conserva recursos**: No malgasta threads/conexiones en llamadas que fallarán

---

## Cuándo usar este patrón

### ✅ USAR cuando:

- Llamadas a servicios externos que pueden fallar completamente
- Dependencias que pueden estar temporalmente no disponibles
- Servicios con alta latencia o timeouts frecuentes
- Protección contra fallos en cascada
- Necesidad de fail-fast en lugar de esperar timeouts

### ❌ NO USAR cuando:

- Llamadas a recursos locales (DB local, caché local)
- Operaciones críticas que deben completarse (transacciones de dinero)
- Servicios internos altamente disponibles con SLA 99.99%
- Llamadas poco frecuentes donde el overhead no se justifica

---

## Implementación en este sistema

### Arquitectura del Sistema

```
┌─────────────────────────────────────────────┐
│  Order Service                              │
│  ┌───────────────────────────────────────┐  │
│  │ Circuit Breaker "userService"         │  │
│  │                                       │  │
│  │ Estado: CLOSED / OPEN / HALF_OPEN    │  │
│  └───────────────┬───────────────────────┘  │
└────────────────────┼──────────────────────────┘
                     │
                     │ HTTP GET /users/{id}
                     ↓
          ┌──────────────────────┐
          │  User Service        │
          │                      │
          └──────────────────────┘
```

### Ubicación del Código

**Framework**: Resilience4j
**Servicio**: Order Service

### Código Implementado

#### 1. Dependencia Maven (`pom.xml`)

```xml
<!-- Resilience4j para Circuit Breaker -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

#### 2. Configuración (`application.yml`)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService: # Nombre del Circuit Breaker
        register-health-indicator: true
        event-consumer-buffer-size: 10
        automatic-transition-from-open-to-half-open-enabled: true

        # Parámetros clave
        failure-rate-threshold: 50 # 50% de fallos para abrir
        minimum-number-of-calls: 5 # Mínimo 5 llamadas para calcular
        permitted-number-of-calls-in-half-open-state: 3 # 3 llamadas de prueba
        sliding-window-size: 10 # Ventana de 10 llamadas
        wait-duration-in-open-state: 10s # Esperar 10s antes de probar
        sliding-window-type: COUNT_BASED # Basado en número de llamadas

management:
  endpoints:
    web:
      exposure:
        include: health,info,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

#### 3. Aplicar @CircuitBreaker (`CartServiceImpl.java`)

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;

@Override
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackFindById")
@Retryable(
    value = {RestClientException.class, Exception.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
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

                        // Llamada protegida por Circuit Breaker
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
                        throw e;  // Circuit Breaker registra el fallo
                    }
                }
                return c;
            })
            .orElseThrow(() -> new CartNotFoundException(
                String.format("Cart with id: %d not found", cartId)));
}
```

#### 4. Método Fallback (ejecutado cuando el circuito está OPEN)

```java
// Fallback para Circuit Breaker
public CartDto fallbackFindById(Integer cartId, Exception e) {
    log.warn("Circuit Breaker OPEN for userService. " +
             "Returning cart {} without user details. Reason: {}",
             cartId, e.getMessage());

    // Degradación elegante: devolver cart sin UserDto
    return this.cartRepository.findById(cartId)
            .map(CartMappingHelper::map)
            .orElseThrow(() -> new CartNotFoundException(
                String.format("Cart with id: %d not found", cartId)));
}
```

---

## Cómo funciona

### Flujo Completo de Transiciones

#### Fase 1: Estado CLOSED (Normal)

```
┌────────────────────────────────────────────────────────────┐
│ Llamadas: [✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅ ✅]                      │
│ Tasa de fallos: 0%                                         │
│ Estado: CLOSED                                             │
│ Acción: Todas las llamadas pasan normalmente              │
└────────────────────────────────────────────────────────────┘
```

#### Fase 2: Servicio comienza a fallar

```
┌────────────────────────────────────────────────────────────┐
│ Llamadas: [✅ ✅ ❌ ✅ ❌ ❌ ✅ ❌ ❌ ❌]                      │
│                                                            │
│ Cálculo:                                                   │
│   - Total llamadas: 10                                     │
│   - Fallos: 6                                              │
│   - Tasa de fallos: 60%                                    │
│   - Threshold: 50%                                         │
│                                                            │
│ Condición: 60% > 50% → 🚨 ABRIR CIRCUITO                  │
└────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────┐
│ Estado cambió: CLOSED → OPEN                               │
│ Timestamp: 10:30:00                                        │
└────────────────────────────────────────────────────────────┘
```

#### Fase 3: Estado OPEN (Circuito Abierto)

```
┌────────────────────────────────────────────────────────────┐
│ Estado: OPEN                                               │
│ Duración: 10:30:00 - 10:30:10 (10 segundos)              │
│                                                            │
│ Request #1 (10:30:01) → ❌ RECHAZADO → Fallback           │
│ Request #2 (10:30:02) → ❌ RECHAZADO → Fallback           │
│ Request #3 (10:30:03) → ❌ RECHAZADO → Fallback           │
│ ...                                                        │
│ Request #N (10:30:09) → ❌ RECHAZADO → Fallback           │
│                                                            │
│ ⚡ Todos rechazados instantáneamente                       │
│ ✅ User Service tiene tiempo de recuperarse                │
└────────────────────────────────────────────────────────────┘
                         │
                         │ wait_duration (10s) transcurrido
                         ↓
┌────────────────────────────────────────────────────────────┐
│ Estado cambió: OPEN → HALF_OPEN                           │
│ Timestamp: 10:30:10                                        │
└────────────────────────────────────────────────────────────┘
```

#### Fase 4: Estado HALF_OPEN (Probando)

```
┌────────────────────────────────────────────────────────────┐
│ Estado: HALF_OPEN                                          │
│ Llamadas permitidas: 3 (permitted_number_of_calls)        │
│                                                            │
│ Request #1 (10:30:11) → 🧪 ENVIADO → ✅ ÉXITO             │
│ Request #2 (10:30:12) → 🧪 ENVIADO → ✅ ÉXITO             │
│ Request #3 (10:30:13) → 🧪 ENVIADO → ✅ ÉXITO             │
│                                                            │
│ Resultado: 3 de 3 exitosas (100%)                         │
│ Decisión: Servicio se recuperó → CERRAR CIRCUITO          │
└────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────┐
│ Estado cambió: HALF_OPEN → CLOSED                         │
│ Timestamp: 10:30:13                                        │
│ Sistema vuelve a la normalidad ✅                          │
└────────────────────────────────────────────────────────────┘
```

#### Fase 4b: Si las pruebas fallan en HALF_OPEN

```
┌────────────────────────────────────────────────────────────┐
│ Estado: HALF_OPEN                                          │
│                                                            │
│ Request #1 (10:30:11) → 🧪 ENVIADO → ✅ ÉXITO             │
│ Request #2 (10:30:12) → 🧪 ENVIADO → ❌ FALLO             │
│ Request #3 (10:30:13) → 🧪 ENVIADO → ❌ FALLO             │
│                                                            │
│ Resultado: 1 de 3 exitosas (33%)                          │
│ Decisión: Aún hay problemas → ABRIR de nuevo              │
└────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────────────────────────────────────┐
│ Estado cambió: HALF_OPEN → OPEN                           │
│ Timestamp: 10:30:13                                        │
│ Esperar otros 10 segundos antes de probar                 │
└────────────────────────────────────────────────────────────┘
```

---

## Parámetros configurables

### Parámetros Clave en `application.yml`

| Parámetro                                      | Valor Actual  | Descripción                                           |
| ---------------------------------------------- | ------------- | ----------------------------------------------------- |
| `failure-rate-threshold`                       | `50`          | Porcentaje de fallos para abrir el circuito (50%)     |
| `minimum-number-of-calls`                      | `5`           | Mínimo de llamadas antes de calcular tasa de fallos   |
| `sliding-window-size`                          | `10`          | Tamaño de la ventana deslizante (últimas 10 llamadas) |
| `wait-duration-in-open-state`                  | `10s`         | Tiempo en estado OPEN antes de pasar a HALF_OPEN      |
| `permitted-number-of-calls-in-half-open-state` | `3`           | Llamadas de prueba en HALF_OPEN                       |
| `sliding-window-type`                          | `COUNT_BASED` | Tipo de ventana (por cantidad, no por tiempo)         |

### Configuración Externa (ConfigMap)

```yaml
# kubernetes-organization/k8s/order-service/configmap.yaml
CB_FAILURE_RATE: "50" # Cambiar threshold a 60%, 70%, etc.
CB_WAIT_DURATION: "10s" # Cambiar a "5s", "30s", etc.
CB_SLIDING_WINDOW: "10" # Cambiar a "20" para ventana más grande
```

**Uso en `application.yml`**:

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

### Ajustes Recomendados por Escenario

#### Para desarrollo/testing:

```yaml
failure-rate-threshold: 50 # Sensible
wait-duration: 5s # Corto para pruebas rápidas
sliding-window-size: 5 # Ventana pequeña
```

#### Para producción estable:

```yaml
failure-rate-threshold: 60 # Más tolerante
wait-duration: 30s # Dar más tiempo de recuperación
sliding-window-size: 20 # Ventana más grande
```

#### Para servicios críticos:

```yaml
failure-rate-threshold: 70 # Muy tolerante
wait-duration: 60s # Mucho tiempo de recuperación
sliding-window-size: 50 # Ventana muy grande
```

---

## Cómo demostrarlo

### Test 1: Ver transición CLOSED → OPEN

```bash
# Paso 1: Verificar estado inicial (CLOSED)
curl http://api.alianzadelamagiaeterna.com/order-service/actuator/health | jq '.components.circuitBreakers'

# Paso 2: Apagar user-service
kubectl scale deployment user-service --replicas=0 -n ecommerce-dev

# Paso 3: Hacer 10 requests para llenar la ventana
for i in {1..10}; do
  curl -s http://api.alianzadelamagiaeterna.com/order-service/api/carts/1 &
done
wait

# Paso 4: Verificar que el circuito se ABRIÓ
curl http://api.alianzadelamagiaeterna.com/order-service/actuator/health | jq '.components.circuitBreakers.details.userService'

# Resultado esperado:
# {
#   "status": "OPEN",
#   "failureRate": "60.0%",
#   ...
# }

# Paso 5: Hacer otro request y ver que falla INMEDIATAMENTE
time curl http://api.alianzadelamagiaeterna.com/order-service/api/carts/1
# Debería responder en < 1 segundo (fail-fast)
```

### Test 2: Ver transición OPEN → HALF_OPEN → CLOSED

```bash
# Continuación del Test 1...

# Paso 6: Esperar 10 segundos (wait_duration)
echo "Esperando 10 segundos para transición a HALF_OPEN..."
sleep 10

# Paso 7: Verificar estado
curl http://api.alianzadelamagiaeterna.com/order-service/actuator/health | jq

# Paso 8: Levantar user-service
kubectl scale deployment user-service --replicas=1 -n ecommerce-dev

# Paso 9: Esperar que el pod esté listo
kubectl wait --for=condition=ready pod -l app=user-service -n ecommerce-dev --timeout=60s

# Paso 10: Hacer 3 requests (llamadas de prueba en HALF_OPEN)
for i in {1..3}; do
  echo "Request $i"
  curl -s http://api.alianzadelamagiaeterna.com/order-service/api/carts/1
  sleep 1
done

# Paso 11: Verificar que el circuito se CERRÓ
curl http://api.alianzadelamagiaeterna.com/order-service/actuator/health | jq '.components.circuitBreakers.details.userService.status'

# Resultado esperado: "CLOSED"
```

### Test 3: Monitorear eventos en tiempo real

```bash
# Terminal 1: Ver eventos del Circuit Breaker
watch -n 1 'curl -s http://api.alianzadelamagiaeterna.com/order-service/actuator/circuitbreakerevents | jq'

# Terminal 2: Generar tráfico
while true; do
  curl -s http://api.alianzadelamagiaeterna.com/order-service/api/carts/1 > /dev/null
  sleep 0.5
done

# Terminal 3: Apagar/Levantar user-service
kubectl scale deployment user-service --replicas=0 -n ecommerce-dev
sleep 20
kubectl scale deployment user-service --replicas=1 -n ecommerce-dev
```

### Test 4: Comparar con/sin Circuit Breaker

```bash
# SIN Circuit Breaker (user-service caído)
# Cada request espera timeout (5s) y hace reintentos (3x)
# Total: ~15 segundos por request

time curl http://api.alianzadelamagiaeterna.com/order-service/api/carts/1
# real 15.234s

# CON Circuit Breaker (después de que se abre)
# Request rechazado inmediatamente, fallback ejecutado
time curl http://api.alianzadelamagiaeterna.com/order-service/api/carts/1
# real 0.123s ⚡ 120x más rápido!
```

---

## Integración con Retry Pattern

### Orden de Ejecución

Cuando ambos patrones están aplicados al mismo método:

```java
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackFindById")  // 1️⃣ Primero
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))             // 2️⃣ Segundo
public CartDto findById(Integer cartId) { }
```

### Flujo de Decisión

```
Request
  ↓
┌─────────────────────────────────┐
│ Circuit Breaker evalúa          │
│ ¿Estado = OPEN?                 │
└────────┬─────────────────┬──────┘
         │                 │
        NO                YES
         │                 │
         ↓                 ↓
┌────────────────┐  ┌──────────────┐
│ CLOSED/HALF    │  │   OPEN       │
│   OPEN         │  │              │
│ Enviar request │  │ Rechazar     │
└───────┬────────┘  │ Ejecutar     │
        │           │ Fallback     │
        ↓           └──────────────┘
┌────────────────────────────┐
│ Retry Pattern actúa        │
│ Si falla: Reintentar       │
│   Intento 1 → Fallo        │
│   Intento 2 → Fallo        │
│   Intento 3 → Éxito        │
└────────┬───────────────────┘
         │
         ↓
┌────────────────────────────┐
│ Circuit Breaker registra   │
│ el resultado (éxito/fallo) │
│ Actualiza métricas         │
└────────────────────────────┘
```

### Comportamiento Combinado

#### Caso 1: Fallos ocasionales (Retry ayuda)

```
Estado CB: CLOSED
Request → Fallo temporal
  ↓
Retry #1 → Éxito ✅
  ↓
CB registra: Éxito
CB permanece: CLOSED
```

#### Caso 2: Servicio completamente caído (CB protege)

```
Request #1-5: CB CLOSED, Retry 3x cada uno → Todos fallan
  ↓
CB detecta: 100% tasa de fallos
CB cambia: CLOSED → OPEN
  ↓
Request #6-100: CB OPEN → Rechazados inmediatamente
  ↓
NO se activa Retry (CB ya rechazó)
✅ Ahorro de 300 reintentos innecesarios
```

### Beneficios de la Combinación

1. **Retry**: Maneja fallos transitorios individuales
2. **Circuit Breaker**: Protege contra fallos sistemáticos
3. **Juntos**: Sistema resiliente que se adapta a diferentes tipos de fallos

---

## Métricas y monitoreo

### Endpoints de Actuator

#### 1. Estado del Circuit Breaker

```bash
curl http://order-service:8080/order-service/actuator/health | jq '.components.circuitBreakers'
```

**Respuesta**:

```json
{
  "status": "UP",
  "details": {
    "userService": {
      "status": "CLOSED",
      "failureRate": "20.0%",
      "slowCallRate": "0.0%",
      "failureRateThreshold": 50.0,
      "slowCallRateThreshold": 100.0,
      "bufferedCalls": 10,
      "failedCalls": 2,
      "slowCalls": 0,
      "slowFailedCalls": 0,
      "notPermittedCalls": 0
    }
  }
}
```

#### 2. Eventos del Circuit Breaker

```bash
curl http://order-service:8080/order-service/actuator/circuitbreakerevents
```

**Respuesta**:

```json
{
  "circuitBreakerEvents": [
    {
      "circuitBreakerName": "userService",
      "type": "STATE_TRANSITION",
      "creationTime": "2025-01-26T10:30:00.123Z",
      "stateTransition": "CLOSED_TO_OPEN",
      "failureRate": "60.0"
    },
    {
      "circuitBreakerName": "userService",
      "type": "STATE_TRANSITION",
      "creationTime": "2025-01-26T10:30:10.456Z",
      "stateTransition": "OPEN_TO_HALF_OPEN"
    }
  ]
}
```

### Métricas clave a monitorear

| Métrica             | Descripción                           | Alerta si...                |
| ------------------- | ------------------------------------- | --------------------------- |
| `failureRate`       | Tasa de fallos actual                 | > 40% (cerca del threshold) |
| `state`             | Estado del CB (CLOSED/OPEN/HALF_OPEN) | OPEN por más de 5 minutos   |
| `notPermittedCalls` | Llamadas rechazadas por CB OPEN       | Aumenta continuamente       |
| `bufferedCalls`     | Llamadas en la ventana deslizante     | < minimum_number_of_calls   |
| `failedCalls`       | Total de llamadas fallidas            | Aumenta rápidamente         |

### Dashboards recomendados (Grafana/Prometheus)

```
Panel 1: Estado del Circuit Breaker (Gauge)
  - CLOSED: Verde
  - HALF_OPEN: Amarillo
  - OPEN: Rojo

Panel 2: Tasa de Fallos (Line Chart)
  - failureRate vs failureRateThreshold

Panel 3: Llamadas por Estado (Stacked Bar)
  - Successful calls
  - Failed calls
  - Not permitted calls

Panel 4: Transiciones de Estado (Timeline)
  - Eventos de STATE_TRANSITION
```

---

## Ventajas y limitaciones

### ✅ Ventajas

1. **Fail-fast**: Respuestas inmediatas cuando el servicio está caído
2. **Previene fallos en cascada**: Protege el sistema completo
3. **Auto-recuperación**: Detecta automáticamente cuando el servicio vuelve
4. **Conserva recursos**: No malgasta threads/conexiones
5. **Protege servicio destino**: Le da tiempo de recuperarse
6. **Métricas incorporadas**: Monitoreo detallado incluido
7. **Configurable**: Ajustable según necesidades

### ⚠️ Limitaciones

1. **Complejidad adicional**: Más lógica que mantener
2. **Configuración delicada**: Parámetros incorrectos pueden causar problemas
3. **Falsos positivos**: Puede abrir en picos temporales
4. **Estado compartido**: Múltiples instancias necesitan coordinación
5. **Latencia inicial**: Primeras llamadas para detectar fallo
6. **Requiere fallbacks**: Necesitas manejar el caso OPEN

### 🚫 Anti-patrones a evitar

1. **Threshold muy bajo**: CB se abre demasiado fácil
2. **Wait duration muy corto**: No da tiempo de recuperación
3. **Sin fallback**: Usuario recibe errores crudos
4. **Ventana muy pequeña**: No suficientes datos para decidir
5. **Ignorar métricas**: No monitorear el estado del CB
6. **Todos los servicios con misma config**: Cada servicio es diferente

---

## Mejores prácticas

### 1. Definir fallbacks apropiados

```java
public CartDto fallbackFindById(Integer cartId, Exception e) {
    // Opción 1: Datos en caché
    return cacheService.getCart(cartId);

    // Opción 2: Degradación elegante
    return CartDto.builder()
        .cartId(cartId)
        .userDto(UserDto.builder().username("Usuario temporal").build())
        .build();

    // Opción 3: Respuesta parcial
    return getCartWithoutUserDetails(cartId);
}
```

### 2. Configurar por tipo de servicio

```yaml
# Servicio crítico (alta disponibilidad)
criticalService:
  failure-rate-threshold: 70
  wait-duration: 60s

# Servicio externo (menos confiable)
externalService:
  failure-rate-threshold: 40
  wait-duration: 30s

# Servicio interno (muy confiable)
internalService:
  failure-rate-threshold: 80
  wait-duration: 10s
```

### 3. Monitorear activamente

```bash
# Alertas en Prometheus
- alert: CircuitBreakerOpen
  expr: circuit_breaker_state{name="userService"} == 1
  for: 5m
  annotations:
    summary: "Circuit Breaker para userService está OPEN"

- alert: HighFailureRate
  expr: circuit_breaker_failure_rate > 0.40
  for: 2m
  annotations:
    summary: "Tasa de fallos alta: {{ $value }}%"
```

### 4. Testear transiciones

```java
@Test
public void testCircuitBreakerTransitions() {
    // Simular 6 fallos para abrir CB
    for (int i = 0; i < 6; i++) {
        assertThrows(Exception.class, () ->
            cartService.findById(1));
    }

    // Verificar que el CB está OPEN
    CircuitBreakerRegistry registry = CircuitBreakerRegistry.ofDefaults();
    CircuitBreaker cb = registry.circuitBreaker("userService");
    assertEquals(CircuitBreaker.State.OPEN, cb.getState());

    // Siguiente llamada debe fallar inmediatamente
    long start = System.currentTimeMillis();
    assertThrows(Exception.class, () -> cartService.findById(1));
    long duration = System.currentTimeMillis() - start;

    assertTrue(duration < 100); // Fail-fast en < 100ms
}
```

### 5. Documentar dependencias

Mantener un mapa de qué Circuit Breakers protegen qué dependencias:

```
order-service:
  - userService CB → USER-SERVICE
  - paymentService CB → PAYMENT-SERVICE
  - inventoryService CB → INVENTORY-SERVICE

user-service:
  - authService CB → AUTH-SERVICE
  - emailService CB → EMAIL-SERVICE
```

### 6. Combinar con otros patrones

```java
@CircuitBreaker(name = "userService")     // Protección contra fallos masivos
@Retryable(maxAttempts = 3)               // Manejo de fallos transitorios
@Timeout(value = 5, unit = TimeUnit.SECONDS)  // Evita esperas infinitas
@RateLimiter(name = "userService")        // Previene sobrecarga
public CartDto findById(Integer cartId) { }
```

### 7. Establecer SLOs claros

```yaml
# Service Level Objectives
userService:
  target_availability: 99.9%
  max_failure_rate: 0.1%
  max_latency_p99: 500ms

# Configurar CB acorde a SLOs
circuit_breaker:
  failure-rate-threshold: 1.0 # 1% (10x el target)
  slow-call-duration-threshold: 1000ms # 2x latency p99
```

---

## Conclusión

El **Circuit Breaker Pattern** es esencial para construir sistemas distribuidos resilientes. Actúa como una "válvula de seguridad" que previene que fallos individuales derriben todo el sistema, mientras permite recuperación automática cuando los servicios vuelven a estar disponibles.

**Integración perfecta con**:

- ✅ **Retry Pattern**: Maneja fallos transitorios antes de que el CB intervenga
- ✅ **Timeout Pattern**: Define cuándo considerar una llamada como "fallida"
- ✅ **Fallback Pattern**: Proporciona respuestas degradadas cuando el CB está OPEN
- ✅ **Externalized Configuration**: Permite ajustes operacionales sin deployar

**Próximos pasos**:

1. Implementar dashboards de monitoreo
2. Definir alertas en base a métricas del CB
3. Documentar runbooks para cuando el CB se abre
4. Realizar chaos engineering para validar comportamiento

---

**Implementado en**: Order Service
**Framework**: Resilience4j
**Versión**: 0.1.0
**Última actualización**: Enero 2025
