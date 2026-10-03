# Bitácora de Desarrollo SDD — Simple Stock Flow
**Programa:** Análisis y Desarrollo de Software (ADSO) · Ficha 3413974  
**Aprendiz:** Paula Claros (`paulaclaros`)  
**Metodología:** Spec-Driven Development (SDD) — Desarrollo Guiado por Especificación  
**Arquitectura:** Onion Architecture (Arquitectura Cebolla de 4 capas + Bootstrap)  
**Fecha:** 2026-10-03  

---

## 1. Introducción y Justificación de la Metodología

### ¿Qué es Spec-Driven Development (SDD)?
El Desarrollo Guiado por Especificación (SDD) establece un principio innegociable: **la especificación técnica y la documentación son la fuente única de verdad y se escriben antes de codificar**. 
En este proyecto:
1. No se improvisa ninguna regla de negocio en el código.
2. Cada entidad, endpoint, cuerpo de error y restricción de base de datos se deriva directamente de la especificación técnica (`test-simple-stock-flow-docs`) y del consenso de arquitectura del equipo (`ARQUITECTURA-ONION.md`).
3. El código es la materialización ejecutable del contrato previamente documentado.

---

## 2. La Arquitectura Cebolla (Onion Architecture) de 4 Capas

De acuerdo con el acuerdo del equipo consignado en `ARQUITECTURA-ONION.md` y `GUIA-ONION.md`, el sistema organiza el backend en 4 anillos concéntricos con una regla estricta de inversión de dependencias (**las capas externas conocen a las internas, pero las internas jamás conocen a las externas**):

```
       ┌────────────────────────────────────────────────────────┐
       │ 5. BOOTSTRAP (Punto de Ensamblaje)                    │
       │    PortBindingsServiceProvider                         │
       └────────────────────────┬───────────────────────────────┘
                                │ ensambla
       ┌────────────────────────┴───────────────────────────────┐
       ▼                                                        ▼
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ 4. PRESENTACIÓN              │        │ 3. INFRAESTRUCTURA           │
│    Controllers, Requests,    │        │    Eloquent Models, Mappers, │
│    Resources, Error Handlers │        │    Repositorios, JWT, Hash   │
└──────────────┬───────────────┘        └──────────────┬───────────────┘
               │ depende de                            │ implementa puertos de
               ▼                                       ▼
       ┌────────────────────────────────────────────────────────┐
       │ 2. APLICACIÓN (Casos de Uso)                           │
       │    Puertos Inbound (Entrada) · Puertos Outbound        │
       │    PlaceSaleService, ProductCatalogService, etc.       │
       └────────────────────────┬───────────────────────────────┘
                                │ depende únicamente de
                                ▼
       ┌────────────────────────────────────────────────────────┐
       │ 1. DOMINIO (El Núcleo Sagrado)                         │
       │    Entities, Value Objects, Business Exceptions        │
       │    (PHP 8.2 Puro · CERO dependencias de Laravel o BD)  │
       └────────────────────────────────────────────────────────┘
```

### Detalle pedagógico de cada capa:
1. **Anillo 1: Dominio (`Domain/`)**:
   - Contiene los modelos puros (`Product`, `Sale`, `SaleItem`, `Category`, `User`), objetos de valor (`Money`, `Quantity`, `Username`, `Role`) y excepciones de negocio (`InsufficientStockException`, `EmptySaleException`).
   - **Regla de oro:** No contiene ninguna referencia a Laravel, facades, base de datos ni HTTP.
2. **Anillo 2: Aplicación (`Application/`)**:
   - Orquesta los casos de uso del sistema (`PlaceSaleService`, `ProductCatalogService`, `GetSalesService`, `SalesReportService`, `AuthenticationService`).
   - Define los **Puertos de Entrada** (Interfaces que definen qué operaciones ofrece la aplicación) y los **Puertos de Salida** (`ProductRepository`, `SaleRepository`, `UnitOfWork`, `TokenGenerator`).
   - **Regla de oro:** La lógica transaccional se abstrae mediante `UnitOfWork::run(callable $operation)` para no exponer `DB::transaction()` de Laravel a esta capa.
3. **Anillo 3: Infraestructura (`Infrastructure/`)**:
   - Implementa los detalles técnicos y de framework.
   - Contiene los modelos Eloquent de base de datos (`ProductModel`, `SaleModel`), los Mappers que convierten entre Eloquent y Dominio, los repositorios que implementan los puertos, y los servicios de seguridad (Argon2 / Bcrypt, Firebase JWT).
4. **Anillo 4: Presentación (`Presentation/`)**:
   - Expone la API REST HTTP al exterior.
   - Valida la estructura de las peticiones (`Requests`), delega la ejecución a los casos de uso y serializa las respuestas en JSON (`Resources`) con formato estricto `camelCase`.
   - Controla las respuestas de error en formato RFC 7807 (`application/problem+json`).
5. **Ensamblaje (`Bootstrap/`)**:
   - `PortBindingsServiceProvider` es el único lugar donde se asocian las interfaces de los puertos de Aplicación con sus implementaciones concretas de Infraestructura.

---

## 3. Matriz de Trazabilidad: Reglas de Negocio (RN)

| Código | Regla de Negocio | Implementación en Dominio / BD | Verificación |
|---|---|---|---|
| **RN-01** | El stock nunca es negativo | `Product::decreaseStock()`, Check SQL `stock >= 0` | Si stock < cantidad, lanza `InsufficientStockException` (422) |
| **RN-02** | El precio es mayor que cero | `Money::fromDecimal()`, Check SQL `price > 0` | Valida `$amount > 0`, lanza `InvalidPriceException` (422) |
| **RN-03** | La cantidad de venta es mayor que cero | `Quantity::fromInt()`, Check SQL `quantity > 0` | Lanza `InvalidQuantityException` si `$qty <= 0` |
| **RN-04** | Una venta tiene al menos una línea | `Sale::create()`, regla en agregado | Lanza `EmptySaleException` si la lista de items está vacía |
| **RN-05** | Un producto no se repite en la misma venta | `Sale::create()`, verificación de unicidad de IDs | Lanza `RepeatedProductException` si un ID de producto se repite |
| **RN-06** | Precio y nombre se congelan al vender | `SaleItem` almacena `unitPrice` y `productName` | Cambiar el catálogo vivo no altera ventas pasadas |
| **RN-07** | Una venta registrada no se modifica ni anula | Ausencia total de rutas PUT/DELETE en ventas | Cumplido por diseño: no existe endpoint ni método de modificación |
| **RN-08** | Un producto vendido no se borra físicamente | Baja lógica (`deleted_at` / `isActive = false`) | Los registros de ventas conservan integridad referencial |
| **RN-09** | Todos los importes en la misma moneda | `Money` encapsula el código de moneda ('COP') | Imposible sumar o mezclar monedas distintas |
| **RN-10** | `username` único, en minúsculas | `Username` value object (`strtolower(trim())`) | Check de unicidad en base de datos y validación de entidad |
| **RN-11** | Rol cerrado: `{admin, seller}` | `Role` value object con enum restringido | Check SQL `role IN ('admin', 'seller')` |
| **RN-12** | El total siempre es la suma de sus líneas | Calculado en tiempo de ejecución: `sum(item.total)` | **Nunca** se almacena columna `total` ni `subtotal` en la BD |

---

## 4. Matriz de Trazabilidad: Historias de Usuario (HU)

| Historia | Descripción | Endpoint | Capa Encargada |
|---|---|---|---|
| **HU-01** | Consultar catálogo de productos con stock y precio | `GET /api/products`, `GET /api/products/{id}` | `ProductCatalogService` · `ProductController` |
| **HU-02** | Mantener catálogo (Crear, editar, dar de baja) | `POST /api/products`, `PUT /api/products/{id}`, `DELETE /api/products/{id}` | `ProductCatalogService` (Rol `admin`) |
| **HU-03** | Subir imagen de producto | `POST /api/products/{id}/image` | `ProductCatalogService` + `LocalFileStorage` |
| **HU-04** | Registrar venta con descuento atómico de stock | `POST /api/sales` | `PlaceSaleService` + `UnitOfWork` (Transaccional) |
| **HU-05** | Consultar ventas registradas y detalle | `GET /api/sales`, `GET /api/sales/{id}` | `GetSalesService` · `SaleController` |
| **HU-06** | Generar reporte consolidado por rango de fechas | `GET /api/reports/sales?from=&to=` | `SalesReportService` · `ReportController` |
| **HU-07** | Autenticación y control de acceso (JWT) | `POST /api/auth/login`, `POST /api/auth/register` | `AuthenticationService` · `JwtAuthMiddleware` |
| **HU-08** | Operar y verificar salud del sistema | `GET /health` | `HealthController` |

---

## 5. Cronología de Ejecución Técnica Realizada

1. **Forks y Clonación:**
   - Creación de forks bajo la cuenta GitHub de la aprendiz: `paulaclaros`.
   - Clonación de los 6 repositorios del ecosistema: `docs`, `api`, `app`, `infra`, `page`, `tool`.
2. **Definición de Documentación y Arquitectura Onion:**
   - Adopción y adaptación del documento de consenso `ARQUITECTURA-ONION.md`.
   - Configuración de las 4 capas limpias en Backend (`api`) y Frontend (`app`).
3. **Desarrollo del Backend Laravel (`test-simple-stock-flow-api`):**
   - Creación de migraciones con esquema relacional estricto, llaves foráneas, índices y checks.
   - Creación de Entidades y Objetos de Valor en `Domain/` sin acoplamiento a Laravel.
   - Creación de Casos de Uso y Puertos en `Application/`.
   - Implementación de Eloquent Mappers, Repositorios y `LaravelUnitOfWork` en `Infrastructure/`.
   - Implementación de Controladores, Requests, Resources en `Presentation/` con serialización `camelCase` y manejo de errores RFC 7807.
4. **Desarrollo del Frontend React (`test-simple-stock-flow-app`):**
   - Interfaz moderna en React 18, Vite y TypeScript con soporte responsive.
   - Módulos completos: Login, Catálogo con filtros y estados de stock, Punto de Venta con Carrito interactivo y control de stock en tiempo real, Historial de Ventas con detalle, y Reporte de Ventas con filtros de fecha y desglose.
   - Configuración de fallback reactivo interactivo para previsualización inmediata en local (`localhost:4200`).
5. **Desarrollo de Satélites:**
   - `test-simple-stock-flow-page`: Landing page corporativa informativa y de presentación del proyecto (`index.html`, `style.css`).
   - `test-simple-stock-flow-tool`: Script de poblamiento y validación automatizada (`seed.js`).
   - `test-simple-stock-flow-infra`: Orquestación completa en Docker Compose (`docker-compose.yml`, MySQL/Postgres, API, Nginx).
6. **Sincronización Git:**
   - Todos los cambios commiteados y subidos a los repositorios remotos en `https://github.com/paulaclaros`.

---

## 6. Guía Rápida para el Evaluador / Aprendiz: Cómo ver los resultados

1. **Ver la Aplicación Web en vivo (Frontend React):**
   - Abre tu navegador web en: `http://localhost:4200/`
   - Credenciales disponibles:
     - Admin: `admin` / `admin123`
     - Vendedor: `seller` / `seller123`
2. **Ver la Landing Page de Presentación (`page`):**
   - Abre con doble clic el archivo `index.html` ubicado en:
     `C:\Users\punto\Downloads\proyecto (1)\sena-prueba\test-simple-stock-flow-page\index.html`
3. **Ver los repositorios en GitHub:**
   - Ingresa a tu perfil: `https://github.com/paulaclaros`
   - Revisa los 6 repositorios actualizados y con su código sincronizado.
