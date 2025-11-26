# Documentación de Servicios

Este documento proporciona una visión general de todos los microservicios que componen el sistema de ecommerce.

## Servicios de Infraestructura

### Service Discovery

**Repositorio**: `service-discovery-organization`

Servidor Netflix Eureka que actúa como registro centralizado de todos los microservicios. Permite que los servicios se registren automáticamente y descubran otros servicios sin necesidad de conocer sus direcciones IP o puertos.

- Puerto: 8761
- Nombre en Eureka: `SERVICE-DISCOVERY`
- Dashboard web disponible en `/`

### Cloud Config

**Repositorio**: `cloud-config-organization`

Servidor Spring Cloud Config que centraliza la configuración de todos los microservicios desde un repositorio Git. Permite gestionar configuraciones por ambiente sin necesidad de recompilar aplicaciones.

- Puerto: 9296
- Nombre en Eureka: `CLOUD-CONFIG`
- Configuración opcional (los servicios funcionan sin él)

### API Gateway

**Repositorio**: `api-gateway-organization`

Punto de entrada único para todas las peticiones HTTP que llegan a los microservicios. Implementa enrutamiento, balanceo de carga, CORS y circuit breaker usando Spring Cloud Gateway.

- Puerto: 8080
- Nombre en Eureka: `API-GATEWAY`
- Rutas configuradas: `/product-service/**`, `/order-service/**`, `/user-service/**`, `/payment-service/**`, `/shipping-service/**`, `/favourite-service/**`, `/app/**`

### Proxy Client

**Repositorio**: `proxy-client-ecommerce`

Aplicación web frontend que actúa como cliente para consumir los microservicios backend. Construida con Spring Boot, Thymeleaf y OpenFeign.

- Puerto: 8900
- Context Path: `/app`
- Nombre en Eureka: `PROXY-CLIENT`
- Accesible a través de API Gateway en `/app/**`

## Servicios de Negocio

### Product Service

**Repositorio**: `product-service-organization`

Gestiona productos y categorías del catálogo. Implementa soft delete para productos y categorías reservadas del sistema.

- Puerto: 8500
- Nombre en Eureka: `PRODUCT-SERVICE`
- Base de datos: H2 (dev) / MySQL (stage/prod)
- Endpoints principales:
  - `/api/products` - CRUD de productos
  - `/api/categories` - CRUD de categorías
- Características especiales:
  - Soft delete de productos
  - Categorías reservadas ("Deleted", "No category")
  - Migración automática de productos al eliminar categoría

### Order Service

**Repositorio**: `order-service-organization`

Gestiona órdenes y carritos de compra. Implementa soft delete para órdenes y gestión de estados de orden.

- Puerto: 8084
- Nombre en Eureka: `ORDER-SERVICE`
- Base de datos: PostgreSQL
- Endpoints principales:
  - `/api/orders` - CRUD de órdenes
  - `/api/carts` - CRUD de carritos
- Características especiales:
  - Soft delete de órdenes
  - Gestión de estados de orden
  - Circuit breaker para resiliencia

### User Service

**Repositorio**: `user-service-organization`

Gestiona usuarios, credenciales, direcciones y tokens de verificación. Implementa autenticación y gestión completa de usuarios.

- Puerto: 8083
- Nombre en Eureka: `USER-SERVICE`
- Base de datos: PostgreSQL
- Endpoints principales:
  - `/api/users` - CRUD de usuarios
  - `/api/credentials` - CRUD de credenciales
  - `/api/addresses` - CRUD de direcciones
  - `/api/verification-tokens` - CRUD de tokens de verificación
- Características especiales:
  - Búsqueda de usuarios por username
  - Gestión de tokens de verificación
  - Circuit breaker para resiliencia

### Payment Service

**Repositorio**: `payment-service-organization`

Gestiona los pagos asociados a las órdenes de compra. Integra con Order Service para obtener información de órdenes.

- Puerto: 8400
- Nombre en Eureka: `PAYMENT-SERVICE`
- Base de datos: H2 (dev) / MySQL (stage/prod)
- Endpoints principales:
  - `/api/payments` - CRUD de pagos
- Características especiales:
  - Estados de pago (NOT_STARTED, IN_PROGRESS, COMPLETED, FAILED, CANCELLED)
  - Integración con Order Service
  - Circuit breaker para resiliencia

### Shipping Service

**Repositorio**: `shipping-service-organization`

Gestiona los items de orden (order items) que representan los productos incluidos en cada orden de compra.

- Puerto: 8600
- Nombre en Eureka: `SHIPPING-SERVICE`
- Base de datos: H2 (dev) / MySQL (stage/prod)
- Endpoints principales:
  - `/api/shippings` - CRUD de order items
- Características especiales:
  - Clave primaria compuesta (orderId + productId)
  - Integración con Order Service y Product Service
  - Circuit breaker para resiliencia

### Favourite Service

**Repositorio**: `favourite-service-organization`

Gestiona los productos favoritos de los usuarios, permitiendo que los usuarios marquen productos como favoritos.

- Puerto: 8800
- Nombre en Eureka: `FAVOURITE-SERVICE`
- Base de datos: H2 (dev) / MySQL (stage/prod)
- Endpoints principales:
  - `/api/favourites` - CRUD de favoritos
- Características especiales:
  - Clave primaria compuesta (userId + productId + likeDate)
  - Integración con User Service y Product Service
  - Manejo de errores al obtener información de otros servicios
  - Circuit breaker para resiliencia

## Comunicación entre Servicios

Todos los servicios se comunican a través del API Gateway y están registrados en Eureka para descubrimiento de servicios. Los servicios de negocio pueden comunicarse entre sí usando RestTemplate o OpenFeign, resolviendo los nombres de servicio a través de Eureka.

## Orden de Arranque Recomendado

1. Service Discovery (debe iniciar primero)
2. Cloud Config (opcional)
3. Microservicios de negocio (Product, Order, User, Payment, Shipping, Favourite)
4. API Gateway
5. Proxy Client

## Configuración de Ambientes

Todos los servicios están configurados para tres ambientes:

- **Dev**: H2 en memoria, configuración local
- **Stage**: MySQL/PostgreSQL, configuración desde Cloud Config
- **Prod**: MySQL/PostgreSQL, configuración desde Cloud Config

Los servicios se despliegan en Kubernetes en el namespace `ecommerce-dev` para todos los ambientes, diferenciándose por tags de imagen Docker:

- Dev: `dev-latest`
- Stage: `stage-latest`
- Prod: `prod-0.1.0`

