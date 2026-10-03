# ADR-005: Adopción de Arquitectura Cebolla (Onion) de 4 Anillos + Bootstrap

## Estado
Aceptado

## Contexto
El enunciado del reto oficial plantea una arquitectura hexagonal de referencia (`adapters/inbound`, `adapters/outbound`). Sin embargo, el equipo de trabajo acordó estructurar la solución bajo la Arquitectura Cebolla (Onion Architecture) concéntrica propuesta por Jeffrey Palermo, adaptada para Laravel y React.

## Decisión
Se estructura el backend en exactamente 4 anillos concéntricos con dirección estricta de dependencias hacia adentro:
1. **Dominio (Anillo 1 - Centro):** Entidades puras, objetos de valor y excepciones de negocio. Cero dependencias externas.
2. **Aplicación (Anillo 2):** Casos de uso y puertos (Inbound y Outbound).
3. **Infraestructura (Anillo 3):** Implementaciones técnicas (Eloquent, Mappers, Repositorios, JWT, Hash, Storage).
4. **Presentación (Anillo 4):** Controladores HTTP, Requests y Resources serializados en camelCase.
5. **Bootstrap (Punto de Ensamblaje):** `PortBindingsServiceProvider` que conecta interfaces con clases concretas.

## Consecuencias
- Desacoplamiento total de las reglas de negocio respecto al framework Laravel y la base de datos.
- Facilidad para probar el dominio y los casos de uso sin necesidad de arrancar el framework.
