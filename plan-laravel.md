# Plan de Implementación Técnico — Simple Stock Flow (Laravel & React)

**Proyecto:** Simple Stock Flow  
**Ficha:** ADSO 3413974  
**Aprendiz:** Paula Claros  
**Stack:** Backend PHP 8.2+ (Laravel 10, Onion Architecture) · Frontend React 18 (TypeScript, Vite) · Infra Docker Compose  

---

## 1. Alcance y Filosofía del Plan

Este plan traduce la especificación técnica original a la solución concreta en Laravel y React, estructurada en los 4 anillos de la Arquitectura Cebolla (Onion) definidos en `ARQUITECTURA-ONION.md`.

---

## 2. Desglose de Fases y Tareas

### Fase P0: Especificación y Contratos Congelados
* [x] **T-01**: Documentación del acuerdo de Arquitectura Onion de 4 capas + Bootstrap (`ARQUITECTURA-ONION.md`).
* [x] **T-02**: Elaboración de registros de decisiones arquitectónicas (ADR-005 a ADR-010).
* [x] **T-03**: Definición de contratos DTO en formato `camelCase` estricto y respuestas de error RFC 7807 (`application/problem+json`).

### Fase P1: Núcleo de Dominio (Anillo 1)
* [x] **T-04**: Modelado de Entidades puras: `Product`, `Sale`, `SaleItem`, `Category`, `User`.
* [x] **T-05**: Implementación de Objetos de Valor: `Money`, `Quantity`, `DateRange`, `Role`, `Username`.
* [x] **T-06**: Implementación de Excepciones de Negocio con mensajes en español: `InsufficientStockException`, `InvalidPriceException`, `RepeatedProductException`, `EmptySaleException`.

### Fase P2: Persistencia e Infraestructura (Anillo 3)
* [x] **T-07**: Migraciones de base de datos relacional con claves foráneas, índices y restricciones `CHECK` para stock positivo, precios mayores a 0 y roles permitidos.
* [x] **T-08**: Modelos Eloquent aislados dentro de `Infrastructure/Persistence/Models/`.
* [x] **T-09**: Mappers de transformación bidireccional entre Eloquent y Dominio (`ProductMapper`, `SaleMapper`).
* [x] **T-10**: Implementación del adaptador transaccional `LaravelUnitOfWork` que envuelve `DB::transaction()`.
* [x] **T-11**: Servicios de seguridad: hashing de contraseñas (`BcryptPasswordHasherService`) y emisión de tokens (`FirebaseJwtTokenService`).

### Fase P3: Casos de Uso y Puertos de Aplicación (Anillo 2)
* [x] **T-12**: Puertos de entrada (`Inbound`): `PlaceSale`, `ManageProducts`, `GetSales`, `GetSalesReport`, `Authenticate`.
* [x] **T-13**: Puertos de salida (`Outbound`): `ProductRepositoryInterface`, `SaleRepositoryInterface`, `UnitOfWork`, `TokenGenerator`, etc.
* [x] **T-14**: Casos de uso implementados:
  * `PlaceSaleService`: Orquestación atómica de venta, validación de stock y congelamiento de precio/nombre.
  * `ProductCatalogService`: Gestión de catálogo, alta, edición y baja lógica.
  * `GetSalesService`: Listado paginado e inspección de ventas.
  * `SalesReportService`: Consolidación de ingresos y unidades vendidas en rango de fechas.
  * `AuthenticationService`: Login con token JWT y registro de usuarios por administradores.

### Fase P4: Capa de Presentación (Anillo 4) y Bootstrap
* [x] **T-15**: Controladores REST en `Presentation/Controllers/` que delegan directamente a los casos de uso.
* [x] **T-16**: Middleware de autenticación JWT (`JwtAuthMiddleware`) y control de roles (`RoleMiddleware`).
* [x] **T-17**: Serializadores y Recursos JSON en formato `camelCase` estricto.
* [x] **T-18**: Punto de ensamblaje en `app/Bootstrap/PortBindingsServiceProvider.php`.

### Fase P5: Frontend React (TypeScript + Vite)
* [x] **T-19**: Estructuración del frontend en 4 capas equivalentes: `domain/`, `application/ports/`, `application/use-cases/`, `infrastructure/http/`.
* [x] **T-20**: Pantalla de Login con gestión de sesión y roles (`admin` y `seller`).
* [x] **T-21**: Catálogo interactivo con filtros, búsqueda y administración de productos.
* [x] **T-22**: Punto de Venta (POS) con Carrito reactivo, cálculo de subtotales y descuento atómico de stock.
* [x] **T-23**: Historial de Ventas y detalle por factura.
* [x] **T-24**: Reporte consolidado con selector de rango de fechas y métricas de ingresos.

### Fase P6: Despliegue y Satélites
* [x] **T-25**: Orquestación en Docker Compose (`infra`).
* [x] **T-26**: Landing page estática institucional (`page`).
* [x] **T-27**: Seeder automatizado (`tool/seed.js`).
