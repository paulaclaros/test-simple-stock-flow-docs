# ADR-007: Manejo Monetario y Precisión Decimal

## Estado
Aceptado

## Contexto
PHP no cuenta con un tipo de dato primitivo `Decimal` como Python o C#. El uso de números de punto flotante (`float`) introduce errores acumulativos de redondeo inaceptables en sistemas contables y de facturación.

## Decisión
Se encapsulan todos los montos de dinero dentro del Value Object inmutable `Money`.
- Se restringe la escala estrictamente a 2 decimales con redondeo `HALF_UP`.
- Se prohíbe el uso de `float` en operaciones aritméticas de negocio.
- En la base de datos se almacena en columnas `DECIMAL(12, 2)`.

## Consecuencias
- Cero discrepancias en centavos en totales y reportes acumulados.
