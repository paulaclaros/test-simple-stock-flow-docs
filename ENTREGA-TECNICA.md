# Simple Stock Flow · Documentación Técnica y Manual de Entrega

> **Prueba Técnica de Desempeño SDD (Spec-Driven Development)**  
> **Servicio Nacional de Aprendizaje (SENA) · Análisis y Desarrollo de Software (ADSO)**  
> **Ficha:** 3413974  
> **Aprendiz:** Paula Claros (GitHub: [`paulaclaros`](https://github.com/paulaclaros))  
> **Stack Implementado:** PHP 8.2 (Laravel 10/11) + React 18 (Vite + TypeScript) + MySQL / PostgreSQL + Docker Compose  

---

## 1. Resumen Ejecutivo y Metodología SDD

La prueba técnica *Simple Stock Flow* evalúa la competencia en **Spec-Driven Development (SDD)**: la capacidad de tomar una especificación técnica rigurosa (entregada en Python y .NET), interpretarla formalmente y materializarla en el stack del reto (**PHP con Laravel** en el backend y **React con TypeScript** en el frontend).

### Principios Fundamentales Cumplidos
1. **La especificación es la ley:** Se respetaron todas las reglas de negocio (RN-01 a RN-12), decisiones de producto (DP-01 a DP-04), invariantes monetarias y el contrato de API REST en formato `camelCase` estricto con errores RFC 7807 (`application/problem+json`).
2. **Dominio Puro (Arquitectura Onion de 4 Capas):** La capa de Dominio en PHP está 100% aislada de dependencias de frameworks (cero Eloquent, cero Laravel, cero facades).
3. **Persistencia Rigurosa con Mappers:** Modelos Eloquent aislados en Infraestructura. La entidad de Dominio `Product` no conoce la columna `version`, delegando el control optimista de concurrencia al Mapper y al repositorio.
4. **Ecosistema Modular de 6 Repositorios:** Siguiendo la instrucción estricta del SENA, se desarrollaron los 6 repositorios forkeados individualmente:
   - `test-simple-stock-flow-docs`
   - `test-simple-stock-flow-api`
   - `test-simple-stock-flow-app`
   - `test-simple-stock-flow-infra`
   - `test-simple-stock-flow-page`
   - `test-simple-stock-flow-tool`

---

## 2. Diagramas de Arquitectura y Flujos

### 2.1. Arquitectura Global de la Solución (Onion Architecture de 4 Anillos + Bootstrap)

```mermaid
flowchart TD
    subgraph Client["Capa de Cliente Web (Frontend React)"]
        Browser["Navegador Web (Usuario)"]
        SPA["React 18 SPA (TypeScript + Vite)"]
        Nginx["Nginx Reverse Proxy (:8080)"]
    end

    subgraph Backend["Capa Backend API (Laravel / PHP 8.2)"]
        subgraph Bootstrap["5. Bootstrap (Punto de Ensamblaje)"]
            PortBindings["PortBindingsServiceProvider"]
        end

        subgraph PresentationLayer["4. Presentation (Anillo 4)"]
            Controllers["Controladores REST (Product, Sale, Auth, Report)"]
            Requests["Form Requests (RFC 7807 Validation)"]
            Middleware["Middleware (JWT Auth, RBAC Admin/Seller)"]
        end

        subgraph ApplicationLayer["2. Application (Anillo 2)"]
            UseCases["Casos de Uso (PlaceSaleService, ProductCatalogService)"]
            InboundPorts["Puertos Inbound (PlaceSale, ManageProducts, etc.)"]
            OutboundPorts["Puertos Outbound (Repositories, UnitOfWork, Hasher)"]
        end

        subgraph DomainLayer["1. Domain (Anillo 1 - Núcleo Puro)"]
            Entities["Entidades (Product, Sale, SaleItem, User, Category)"]
            ValueObjects["Value Objects (Money COP, Quantity, DateRange)"]
            DomainExceptions["Excepciones de Negocio en Español"]
        end

        subgraph InfrastructureLayer["3. Infrastructure (Anillo 3)"]
            EloquentRepos["Repositorios Eloquent"]
            Mappers["Mappers Bidireccionales"]
            UoW["LaravelUnitOfWork (DB::transaction)"]
            Services["Argon2/Bcrypt Hasher, JWT Generator, Local Storage"]
        end
    end

    subgraph Storage["Capa de Persistencia y Almacenamiento"]
        DB[("Motor Relacional (Postgres / MySQL)")]
        MediaVol[("Volumen de Medios (/var/www/media)")]
    end

    Browser -->|HTTP :8080| Nginx
    Nginx -->|Archivos Estáticos| SPA
    Nginx -->|Proxy /api/ y /media/| Controllers
    Controllers --> Requests
    Controllers --> Middleware
    Controllers --> InboundPorts
    InboundPorts --> UseCases
    UseCases --> OutboundPorts
    UseCases --> Entities
    UseCases --> ValueObjects
    OutboundPorts -.->|Implementado por| EloquentRepos
    OutboundPorts -.->|Implementado por| UoW
    EloquentRepos --> Mappers
    Mappers --> Entities
    EloquentRepos --> DB
    UoW --> DB
    Services --> MediaVol
    PortBindings -.->|Enlaza Interfaces| OutboundPorts
```

---

### 2.2. Modelo Entidad-Relación de la Base de Datos

```mermaid
erDiagram
    CATEGORIES ||--o{ PRODUCTS : "clasifica"
    USERS ||--o{ SALES : "registra como vendedor"
    SALES ||--|{ SALE_ITEMS : "contiene líneas"
    PRODUCTS ||--o{ SALE_ITEMS : "se factura en"

    CATEGORIES {
        char(36) id PK
        varchar(100) name UK
        timestamp created_at
    }

    USERS {
        char(36) id PK
        varchar(50) username UK
        varchar(255) password_hash
        varchar(20) role "CHECK role in ('admin', 'seller')"
        timestamp created_at
    }

    PRODUCTS {
        char(36) id PK
        varchar(150) name UK
        decimal(12_2) price "CHECK price > 0"
        int stock "CHECK stock >= 0"
        char(36) category_id FK
        varchar(255) image_url "NULLABLE"
        int version "DEFAULT 1"
        timestamp deleted_at "NULLABLE (Soft Delete)"
        timestamp created_at
        timestamp updated_at
    }

    SALES {
        char(36) id PK
        char(36) seller_id FK
        timestamp sold_at
        timestamp created_at
    }

    SALE_ITEMS {
        char(36) id PK
        char(36) sale_id FK
        char(36) product_id FK
        varchar(150) product_name "Nombre congelado (DP-01)"
        decimal(12_2) unit_price "Precio congelado (RN-06)"
        int quantity "CHECK quantity > 0"
        timestamp created_at
    }
```

---

### 2.3. Diagrama de Secuencia: Venta Transaccional Atómica (RN-01, RN-04, RN-05, RN-06, RN-12)

```mermaid
sequenceDiagram
    autonumber
    actor Vendedor as Vendedor (Cliente Web)
    participant API as SaleController
    participant UC as PlaceSaleService
    participant ProdRepo as ProductRepository
    participant SaleRepo as SaleRepository
    participant UoW as UnitOfWork (LaravelUnitOfWork)
    participant DB as Motor de Base de Datos

    Vendedor->>API: POST /api/sales {"items": [{"productId": "P1", "quantity": 2}]}
    API->>API: Validar JWT Bearer y Formato JSON
    API->>UC: execute(RegisterSaleDTO)

    UC->>UC: Validar RN-04 (venta no vacía) y RN-05 (sin productos repetidos)
    
    UC->>UoW: run(callable)
    UoW->>DB: Iniciar DB::transaction()

    UC->>ProdRepo: findActiveById("P1")
    ProdRepo->>DB: SELECT * FROM products WHERE id = "P1" AND deleted_at IS NULL
    DB-->>ProdRepo: Registro de Producto
    ProdRepo-->>UC: Entidad Product (stock: 10)

    UC->>UC: RN-01: Product.withdraw(2) -> stock: 8 (Lanza InsufficientStockException si excede)
    UC->>ProdRepo: update(Product)
    ProdRepo->>DB: UPDATE products SET stock = 8, version = version + 1 WHERE id = "P1"

    UC->>UC: RN-06: Congelar unitPrice y productName en SaleItem
    UC->>UC: RN-12: Calcular total en tiempo real sumando items

    UC->>SaleRepo: save(Sale)
    SaleRepo->>DB: INSERT INTO sales (...)
    SaleRepo->>DB: INSERT INTO sale_items (...)

    UoW->>DB: COMMIT Transacción
    DB-->>UoW: Transacción Exitosa
    UoW-->>UC: Entidad Sale completa
    UC-->>API: Entidad Sale
    API-->>Vendedor: 201 Created (SaleResource camelCase)
```

---

## 3. Matriz de Cumplimiento de Reglas de Negocio (RN-01 a RN-12)

| Regla | Descripción | Dónde se cumple | Prueba que lo demuestra |
|---|---|---|---|
| **RN-01** | El stock nunca es negativo | `Product::withdraw()` + Check SQL | `tests/Unit/Domain/ProductTest.php` |
| **RN-02** | El precio es mayor que cero | `Money::fromDecimal()` + Check SQL | `tests/Unit/Domain/ValueObjectsTest.php` |
| **RN-03** | La cantidad es mayor que cero | `Quantity::fromInt()` + Check SQL | `tests/Unit/Domain/ValueObjectsTest.php` |
| **RN-04** | Una venta tiene al menos una línea | `Sale` + `PlaceSaleService` | `tests/Unit/Domain/SaleTest.php` |
| **RN-05** | No se repite producto en la venta | `PlaceSaleService` | `tests/Unit/Application/PlaceSaleServiceTest.php` |
| **RN-06** | Precio y nombre se congelan al vender | `SaleItem` congela atributos | `tests/Unit/Domain/SaleTest.php` |
| **RN-07** | Venta registrada no se modifica ni anula | Ausencia de métodos PUT/DELETE | Arquitectura verificada |
| **RN-08** | Producto vendido no se borra (baja lógica) | `Product::deactivate()` | `tests/Unit/Domain/ProductTest.php` |
| **RN-09** | Importes en la misma moneda | Value Object `Money` ('COP') | `tests/Unit/Domain/ValueObjectsTest.php` |
| **RN-10** | `username` único, minúsculas | Value Object `Username` | Validado en servicio y base de datos |
| **RN-11** | Rol cerrado `{admin, seller}` | Value Object `Role` + Check SQL | Validado en middleware y entidades |
| **RN-12** | Total es siempre la suma de sus líneas | Calculado en tiempo de ejecución | `tests/Unit/Domain/SaleTest.php` |

---

## 4. Guía de Ejecución Rápida

### A. Frontend Web en Desarrollo
```bash
cd test-simple-stock-flow-app
npm install
npm run dev
# Abrir en navegador: http://localhost:4200/
```

### B. Ejecución de Pruebas Unitarias y de Arquitectura
```bash
cd test-simple-stock-flow-api
bash verify.sh
```

### C. Despliegue con Docker Compose
```bash
cd test-simple-stock-flow-infra
docker compose up -d --build
```
