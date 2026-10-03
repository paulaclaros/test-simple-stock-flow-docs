# ADR-009: Desacoplamiento de Eloquent, Facades y Auto-wiring de Laravel

## Estado
Aceptado

## Contexto
Laravel fomenta el patrón Active Record a través de Eloquent, donde los modelos extienden de `Illuminate\Database\Eloquent\Model` y mezclan persistencia, eventos de base de datos y lógica de negocio. Esto rompe la pureza de la Arquitectura Cebolla.

## Decisión
- Las entidades de Dominio (`Product`, `Sale`, `User`) son clases PHP 8.2 puras, sin herencia de Eloquent.
- Los modelos Eloquent (`ProductModel`, `SaleModel`, etc.) quedan confinados exclusivamente dentro de `Infrastructure/Persistence/Models/`.
- Se implementan `Mappers` dedicados (`ProductMapper`, `SaleMapper`) para la transformación bidireccional entre Eloquent y Dominio.
- La columna `version` (utilizada para control de concurrencia optimista) es gestionada por el Mapper y la infraestructura, sin contaminar la entidad de Dominio.

## Consecuencias
- Si el motor de persistencia cambia en el futuro, el Dominio permanece 100% inalterado.
