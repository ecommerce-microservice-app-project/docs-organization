# Patrones de Diseño - Proyecto Final IngeSoft V

## Resumen

Este documento identifica y documenta los **patrones de diseño de arquitectura de microservicios y nube** utilizados en el proyecto. El enfoque está en patrones arquitectónicos que resuelven problemas específicos de sistemas distribuidos y microservicios, no en patrones de código tipo Gang of Four (Singleton, Factory, etc.).

Los patrones están organizados en tres categorías principales:
- **Patrones de Arquitectura de Microservicios**: Patrones que definen la estructura y comunicación entre servicios
- **Patrones de Resiliencia**: Patrones que mejoran la tolerancia a fallos del sistema
- **Patrones de Configuración**: Patrones que gestionan la configuración en sistemas distribuidos

---

## Patrones de Arquitectura de Microservicios

### 1. API Gateway Pattern

**Descripción**: El API Gateway actúa como punto de entrada único para todos los clientes, enrutando las solicitudes a los microservicios apropiados.

**Implementación**: Spring Cloud Gateway

**Ubicación**: `api-gateway-organization`

**Configuración**:
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: PRODUCT-SERVICE
        uri: lb://PRODUCT-SERVICE
        predicates:
        - Path=/product-service/**
```

**Propósito**:
- Centralizar el punto de entrada de la aplicación
- Enrutar solicitudes a los microservicios correctos
- Implementar cross-cutting concerns (CORS, timeouts, circuit breakers)
- Ocultar la complejidad de la arquitectura de microservicios a los clientes

**Beneficios**:
- Separación de responsabilidades entre clientes y servicios
- Facilita la implementación de políticas de seguridad centralizadas
- Permite agregación de respuestas de múltiples servicios
- Simplifica el manejo de CORS y autenticación

**Ejemplo de Uso**:
```
Cliente → API Gateway (puerto 8080) → Product Service (puerto 8500)
```

---

### 2. Service Discovery Pattern

**Descripción**: Los microservicios se registran automáticamente en un registro centralizado y pueden descubrir otros servicios sin conocer sus direcciones físicas.

**Implementación**: Netflix Eureka Server

**Ubicación**: `service-discovery-organization`

**Configuración en Microservicios**:
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://service-discovery.ecommerce-dev.svc.cluster.local:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
```

**Propósito**:
- Eliminar la necesidad de hardcodear direcciones IP/puertos de servicios
- Facilitar el escalado horizontal (múltiples instancias del mismo servicio)
- Permitir que los servicios se muevan dinámicamente en la infraestructura
- Habilitar load balancing automático

**Beneficios**:
- Desacoplamiento entre servicios
- Escalabilidad mejorada
- Resiliencia ante fallos (si un servicio falla, se elimina del registro)
- Facilita el despliegue en Kubernetes

**Flujo**:
```
1. Microservicio inicia → Se registra en Eureka
2. Otro servicio necesita comunicarse → Consulta Eureka
3. Eureka devuelve lista de instancias disponibles
4. Cliente (o API Gateway) hace load balancing entre instancias
```

---

### 3. External Configuration Pattern

**Descripción**: La configuración de los microservicios se almacena externamente, permitiendo cambios sin recompilar o redesplegar la aplicación.

**Implementación**: Spring Cloud Config Server

**Ubicación**: `cloud-config-organization`

**Configuración**:
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/SelimHorri/cloud-config-server
          clone-on-start: true
```

**Uso en Microservicios**:
```yaml
spring:
  config:
    import: ${SPRING_CONFIG_IMPORT:optional:configserver:http://localhost:9296}
```

**Propósito**:
- Centralizar la configuración de todos los microservicios
- Permitir diferentes configuraciones por ambiente (dev, stage, prod)
- Facilitar cambios de configuración sin redesplegar
- Versionar la configuración mediante Git

**Beneficios**:
- Separación de código y configuración
- Facilita la gestión de múltiples ambientes
- Permite actualización dinámica de configuración
- Historial completo de cambios mediante Git

**Ejemplo**:
```
Cloud Config Server (puerto 9296)
  ↓
Lee configuración desde Git
  ↓
Microservicios consultan configuración al iniciar
```

---

## Patrones de Resiliencia

### 4. Circuit Breaker Pattern (Patrón de Resiliencia)

**Descripción**: Previene que un servicio continúe intentando ejecutar una operación que probablemente fallará, permitiendo que el sistema se recupere más rápido.

**Implementación**: Resilience4j

**Ubicación**: Todos los microservicios

**Configuración**:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        register-health-indicator: true
        failure-rate-threshold: 50
        minimum-number-of-calls: 5
        sliding-window-size: 10
        wait-duration-in-open-state: 5s
        sliding-window-type: COUNT_BASED
```

**Estados del Circuit Breaker**:
- **CLOSED**: Funcionamiento normal, todas las llamadas pasan
- **OPEN**: Demasiados fallos, las llamadas se rechazan inmediatamente
- **HALF_OPEN**: Estado de prueba, permite algunas llamadas para verificar si el servicio se recuperó

**Propósito**:
- Prevenir cascadas de fallos cuando un servicio dependiente está caído
- Reducir la carga en servicios que están fallando
- Proporcionar fallback rápido en lugar de timeouts largos
- Permitir que los servicios se recuperen automáticamente

**Beneficios**:
- Mejora la resiliencia del sistema
- Reduce la latencia en caso de fallos
- Previene sobrecarga en servicios que ya están fallando
- Proporciona métricas de salud del sistema

**Ejemplo de Uso**:
```
Product Service → Order Service (con Circuit Breaker)
Si Order Service falla 5 veces → Circuit Breaker se abre
Llamadas siguientes se rechazan inmediatamente
Después de 5 segundos → Circuit Breaker entra en HALF_OPEN
Si la siguiente llamada tiene éxito → Circuit Breaker se cierra
```

---

## Patrones de Configuración

### 5. External Configuration Pattern (Ya documentado arriba)

Este patrón ya fue documentado en la sección de "Patrones de Arquitectura" como "External Configuration Pattern". Es un patrón fundamental de configuración en microservicios que permite gestionar configuraciones centralizadas y por ambiente.

---

## Patrones de Código (Secundarios)

Los siguientes patrones son patrones de código que complementan la arquitectura, pero no son el enfoque principal del requisito. Se documentan para completitud:

### 6. Repository Pattern

**Descripción**: Abstrae la lógica de acceso a datos, proporcionando una interfaz más orientada a objetos para acceder a la base de datos.

**Implementación**: Spring Data JPA

**Ubicación**: Todos los microservicios con persistencia

**Ejemplo**:
```java
public interface ProductRepository extends JpaRepository<Product, Integer> {
    @Query("SELECT p FROM Product p WHERE p.category.categoryTitle <> 'Deleted'")
    List<Product> findAllWithoutDeleted();
    
    @Query("SELECT p FROM Product p WHERE p.id = :productId AND p.category.categoryTitle <> 'Deleted'")
    Optional<Product> findByIdWithoutDeleted(Integer productId);
}
```

**Propósito**:
- Separar la lógica de negocio de la lógica de acceso a datos
- Proporcionar una abstracción sobre la capa de persistencia
- Facilitar el testing (fácil de mockear)
- Centralizar queries personalizadas

**Beneficios**:
- Código más limpio y mantenible
- Facilita el cambio de tecnología de persistencia
- Mejora la testabilidad
- Reduce el acoplamiento entre capas

**Uso en Servicios**:
```java
@Service
public class ProductServiceImpl implements ProductService {
    private final ProductRepository productRepository;
    
    public List<ProductDto> findAll() {
        return productRepository.findAllWithoutDeleted()
            .stream()
            .map(ProductMappingHelper::map)
            .collect(Collectors.toList());
    }
}
```

---

### 7. DTO (Data Transfer Object) Pattern

**Descripción**: Objetos que transportan datos entre procesos o capas de la aplicación, sin lógica de negocio.

**Implementación**: Clases DTO con Lombok

**Ubicación**: Todos los microservicios

**Ejemplo**:
```java
@NoArgsConstructor
@AllArgsConstructor
@Data
@Builder
public class ProductDto implements Serializable {
    private Integer productId;
    private String productTitle;
    private String imageUrl;
    private String sku;
    private Double priceUnit;
    private Integer quantity;
    private CategoryDto categoryDto;
}
```

**Propósito**:
- Separar la estructura de datos de la entidad de dominio
- Controlar qué datos se exponen en la API
- Reducir el número de llamadas al combinar datos relacionados
- Proteger la estructura interna de las entidades

**Beneficios**:
- Mejor control sobre la API expuesta
- Facilita la evolución de la entidad sin afectar la API
- Permite versionado de APIs
- Mejora el rendimiento al transferir solo datos necesarios

**Flujo**:
```
Entidad (Product) → DTO (ProductDto) → JSON (API Response)
```

---

### 8. Builder Pattern

**Descripción**: Proporciona una forma flexible de construir objetos complejos paso a paso.

**Implementación**: Lombok @Builder

**Ubicación**: Todas las clases de dominio y DTOs

**Ejemplo**:
```java
Product product = Product.builder()
    .productId(1)
    .productTitle("Laptop ASUS")
    .imageUrl("https://example.com/laptop.jpg")
    .sku("LAP-ASUS-001")
    .priceUnit(1299.99)
    .quantity(50)
    .category(category)
    .build();
```

**Propósito**:
- Simplificar la creación de objetos con muchos parámetros
- Hacer el código más legible
- Permitir construcción inmutables
- Facilitar la creación de objetos opcionales

**Beneficios**:
- Código más legible y mantenible
- Reduce errores al construir objetos
- Facilita la creación de objetos de prueba
- Soporta parámetros opcionales de forma clara

---

### 9. Dependency Injection Pattern

**Descripción**: Las dependencias se inyectan en lugar de ser creadas directamente por la clase que las usa.

**Implementación**: Spring Framework IoC Container

**Ubicación**: Toda la aplicación

**Ejemplo**:
```java
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    // Spring inyecta automáticamente las dependencias
}
```

**Anotaciones Utilizadas**:
- `@Service`: Marca una clase como servicio
- `@RestController`: Marca una clase como controlador REST
- `@Repository`: Marca una clase como repositorio
- `@Component`: Componente genérico de Spring
- `@Autowired`: Inyección de dependencias (implícita con @RequiredArgsConstructor)

**Propósito**:
- Reducir el acoplamiento entre clases
- Facilitar el testing (fácil de mockear dependencias)
- Centralizar la gestión del ciclo de vida de objetos
- Mejorar la mantenibilidad del código

**Beneficios**:
- Código más testeable
- Menor acoplamiento
- Facilita cambios en implementaciones
- Mejora la organización del código

---

### 10. Service Layer Pattern

**Descripción**: Separa la lógica de negocio de la capa de presentación y persistencia mediante una capa de servicios.

**Implementación**: Interfaces de servicio con implementaciones

**Ubicación**: Todos los microservicios

**Estructura**:
```
Resource (Controller) → Service Interface → Service Implementation → Repository
```

**Ejemplo**:
```java
// Interface
public interface ProductService {
    List<ProductDto> findAll();
    ProductDto findById(final Integer productId);
    ProductDto save(final ProductDto productDto);
}

// Implementation
@Service
public class ProductServiceImpl implements ProductService {
    // Implementación de la lógica de negocio
}
```

**Propósito**:
- Separar la lógica de negocio de la capa de presentación
- Facilitar el testing de la lógica de negocio
- Permitir múltiples implementaciones
- Centralizar la lógica de negocio

**Beneficios**:
- Mejor organización del código
- Facilita el testing
- Permite reutilización de lógica
- Mejora la mantenibilidad

---

### 11. Exception Handler Pattern

**Descripción**: Manejo centralizado de excepciones mediante un componente global que captura y procesa todas las excepciones.

**Implementación**: `@ControllerAdvice` de Spring

**Ubicación**: Todos los microservicios

**Ejemplo**:
```java
@ControllerAdvice
@Slf4j
@RequiredArgsConstructor
public class ApiExceptionHandler {
    
    @ExceptionHandler(value = {
        MethodArgumentNotValidException.class,
        HttpMessageNotReadableException.class,
    })
    public ResponseEntity<ExceptionMsg> handleValidationException(final T e) {
        // Manejo de excepciones de validación
        return new ResponseEntity<>(ExceptionMsg.builder()
            .msg(e.getBindingResult().getFieldError().getDefaultMessage())
            .httpStatus(HttpStatus.BAD_REQUEST)
            .build(), HttpStatus.BAD_REQUEST);
    }
    
    @ExceptionHandler(value = {
        ProductNotFoundException.class,
    })
    public ResponseEntity<ExceptionMsg> handleApiRequestException(final T e) {
        // Manejo de excepciones de negocio
    }
}
```

**Propósito**:
- Centralizar el manejo de excepciones
- Proporcionar respuestas consistentes de error
- Evitar código duplicado de manejo de excepciones
- Mejorar la experiencia del cliente con mensajes claros

**Beneficios**:
- Código más limpio (sin try-catch en cada método)
- Respuestas de error consistentes
- Facilita el logging centralizado
- Mejora la mantenibilidad

---

### 12. REST Client Pattern

**Descripción**: Abstracción para realizar llamadas HTTP a servicios REST externos de forma declarativa.

**Implementación**: RestTemplate y OpenFeign

**Ubicación**: 
- RestTemplate: `shipping-service`, `payment-service`, `order-service`
- OpenFeign: `proxy-client-ecommerce`

**Ejemplo con RestTemplate**:
```java
@Service
public class OrderItemServiceImpl {
    private final RestTemplate restTemplate;
    
    public OrderItemDto findById(OrderItemId orderItemId) {
        OrderItem orderItem = orderItemRepository.findById(orderItemId)
            .orElseThrow(...);
        
        ProductDto product = restTemplate.getForObject(
            AppConstant.DiscoveredDomainsApi.PRODUCT_SERVICE_API_URL + "/" + productId,
            ProductDto.class
        );
        
        OrderDto order = restTemplate.getForObject(
            AppConstant.DiscoveredDomainsApi.ORDER_SERVICE_API_URL + "/" + orderId,
            OrderDto.class
        );
    }
}
```

**Ejemplo con OpenFeign**:
```java
@FeignClient(name = "PRODUCT-SERVICE", 
             contextId = "productClientService", 
             path = "/product-service/api/products")
public interface ProductClientService {
    
    @GetMapping
    ResponseEntity<ProductProductServiceCollectionDtoResponse> findAll();
    
    @GetMapping("/{productId}")
    ResponseEntity<ProductDto> findById(@PathVariable("productId") String productId);
}
```

**Propósito**:
- Simplificar la comunicación entre microservicios
- Abstraer los detalles de HTTP
- Facilitar el testing (mockeable)
- Proporcionar integración con Service Discovery

**Beneficios**:
- Código más limpio y declarativo
- Integración automática con Eureka
- Facilita el testing
- Reduce el código boilerplate

**Diferencia entre RestTemplate y OpenFeign**:
- **RestTemplate**: Programático, más control, más código
- **OpenFeign**: Declarativo, menos código, integración automática con Eureka

---

### 13. Mapping Helper Pattern

**Descripción**: Clases helper estáticas que centralizan la lógica de mapeo entre entidades de dominio y DTOs.

**Implementación**: Clases helper con métodos estáticos

**Ubicación**: Todos los microservicios

**Ejemplo**:
```java
public interface ProductMappingHelper {
    
    public static ProductDto map(final Product product) {
        return ProductDto.builder()
            .productId(product.getProductId())
            .productTitle(product.getProductTitle())
            .imageUrl(product.getImageUrl())
            .sku(product.getSku())
            .priceUnit(product.getPriceUnit())
            .quantity(product.getQuantity())
            .categoryDto(CategoryDto.builder()
                .categoryId(product.getCategory().getCategoryId())
                .categoryTitle(product.getCategory().getCategoryTitle())
                .imageUrl(product.getCategory().getImageUrl())
                .build())
            .build();
    }
    
    public static Product map(final ProductDto productDto) {
        return Product.builder()
            .productId(productDto.getProductId())
            .productTitle(productDto.getProductTitle())
            // ... mapeo inverso
            .build();
    }
}
```

**Propósito**:
- Centralizar la lógica de mapeo
- Evitar código duplicado
- Facilitar el mantenimiento cuando cambian las estructuras
- Proporcionar mapeo bidireccional consistente

**Beneficios**:
- Código más mantenible
- Consistencia en el mapeo
- Facilita cambios en estructuras
- Mejora la testabilidad

**Uso**:
```java
// Entidad → DTO
ProductDto dto = ProductMappingHelper.map(product);

// DTO → Entidad
Product entity = ProductMappingHelper.map(productDto);
```

---

## Resumen de Patrones Identificados

### Patrones Principales (Arquitectura de Microservicios y Nube)

| Categoría | Patrón | Implementación | Estado |
|-----------|--------|----------------|--------|
| Arquitectura | API Gateway | Spring Cloud Gateway | Implementado |
| Arquitectura | Service Discovery | Netflix Eureka | Implementado |
| Configuración | External Configuration | Spring Cloud Config | Implementado |
| Resiliencia | Circuit Breaker | Resilience4j | Implementado |

### Patrones Secundarios (Código)

| Categoría | Patrón | Implementación | Estado |
|-----------|--------|----------------|--------|
| Estructural | Repository | Spring Data JPA | Implementado |
| Estructural | DTO | Clases DTO con Lombok | Implementado |
| Estructural | Builder | Lombok @Builder | Implementado |
| Comportamiento | Dependency Injection | Spring IoC | Implementado |
| Comportamiento | Service Layer | Interfaces + Implementaciones | Implementado |
| Comportamiento | Exception Handler | @ControllerAdvice | Implementado |
| Integración | REST Client | RestTemplate / OpenFeign | Implementado |
| Integración | Mapping Helper | Clases helper estáticas | Implementado |

**Nota**: Los patrones principales son los que cumplen con el requisito del proyecto. Los patrones secundarios son complementarios y demuestran buenas prácticas de código, pero no son el enfoque principal del requisito.

---

## Diagrama de Patrones en la Arquitectura

```
                    Cliente
                      |
                      ↓
              ┌───────────────┐
              │ API Gateway   │ ← API Gateway Pattern
              │   Pattern     │
              └───────┬───────┘
                      |
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Product  │  │  Order   │  │  User   │
  │ Service  │  │ Service  │  │ Service │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │
       │ Service Discovery Pattern (Eureka)
       │             │             │
       ↓             ↓             ↓
  ┌─────────────────────────────────┐
  │   Repository Pattern (JPA)       │
  │   + DTO Pattern                  │
  │   + Service Layer Pattern        │
  │   + Dependency Injection         │
  └─────────────────────────────────┘
```

---

## Referencias

- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)
- [Netflix Eureka](https://github.com/Netflix/eureka)
- [Spring Cloud Config](https://spring.io/projects/spring-cloud-config)
- [Resilience4j](https://resilience4j.readme.io/)
- [Spring Data JPA](https://spring.io/projects/spring-data-jpa)
- [OpenFeign](https://spring.io/projects/spring-cloud-openfeign)
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns)

---

**Última actualización**: Enero 2025  
**Versión del documento**: 1.0

