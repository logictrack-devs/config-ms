# -logitrack-config-ms
# Puerta de Enlace Centralizada (api-gateway)

Microservicio desarrollado sobre **Spring Cloud Gateway (WebFlux)** que actúa como punto de entrada único (Single Point of Entry) para todas las aplicaciones clientes de LogiTrack.

## Funcionalidades Mínimas Implementadas
*   **Punto de Entrada Único:** Centraliza las peticiones en el puerto `8066`, habilitando el intercambio de recursos de origen cruzado (**CORS**) para aplicaciones frontend (puertos `5173` y `3000`).
*   **Cliente de Descubrimiento:** Se registra dinámicamente como cliente en `eureka-server` (`http://localhost:8065/eureka/`).
*   **Enrutamiento Dinámico y Reescritura:** Redirecciona el tráfico mediante balanceo de carga (`lb://`) utilizando el filtro `StripPrefix=2` para limpiar los prefijos de red antes de llegar a los microservicios destino:
    *   `/applications/apisecurity/**` ➔ `lb://SECURITY-SERVICE`
    *   `/applications/apishipment/**` ➔ `lb://SHIPMENT-SERVICE`
    *   `/applications/apicourier/**` ➔ `lb://COURIER-SERVICE`
*   **Gestión Centralizada de Seguridad:** Intercepta las cabeceras `Authorization` para validar tokens JWT, delegando la autenticación a `security-service` y protegiendo las rutas según los roles (`ADMIN`, `CLIENTE`, `REPARTIDOR`).
*   **Monitorización:** Exposición de endpoints de salud y rutas activas mediante Spring Boot Actuator (`/actuator/health`, `/actuator/gateway`).

## Configuración Principal (`application.yml`)
```yaml
server:
  port: 8066

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: \${EUREKA_URL:http://localhost:8065/eureka/}

management:
  endpoints:
    web:
      exposure:
        include: health,gateway

spring:
  cloud:
    gateway:
      server:
        webflux:
          globalcors:
            add-to-simple-url-handler-mapping: true
            cors-configurations:
              '[/**]':
                allowedOrigins: \${CORS_ALLOWED_ORIGINS:http://localhost:5173,http://localhost:3000}
                allowedMethods: [GET, POST, PUT, PATCH, DELETE, OPTIONS]
                allowedHeaders: [Authorization, Content-Type, Accept]
                maxAge: 3600
          discovery:
            locator:
              enabled: false
          routes:
            - id: security
              uri: lb://SECURITY-SERVICE
              predicates:
                - Path=/applications/apisecurity/**
              filters:
                - StripPrefix=2
            - id: shipment
              uri: lb://SHIPMENT-SERVICE
              predicates:
                - Path=/applications/apishipment/**
              filters:
                - StripPrefix=2
            - id: courier
              uri: lb://COURIER-SERVICE
              predicates:
                - Path=/applications/apicourier/**
              filters:
                - StripPrefix=2
```
