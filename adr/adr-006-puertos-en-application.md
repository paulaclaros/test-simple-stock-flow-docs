# ADR-006: Ubicación de Puertos en Application y no en Domain

## Estado
Aceptado

## Contexto
En la arquitectura Onion clásica de Palermo, las interfaces de repositorio a veces se ubican dentro de Domain. Sin embargo, surge debate en el equipo respecto al Principio de Inversión de Dependencias (DIP) y la pureza del núcleo.

## Decisión
Los puertos (interfaces de repositorios, almacenamiento, hashing, tokens y transaccionalidad `UnitOfWork`) se ubican formalmente en `Application/Ports/Outbound/` y los contratos de casos de uso en `Application/Ports/Inbound/`.
`Domain` permanece 100% puro, enfocado exclusivamente en entidades, invariantes y objetos de valor.

## Consecuencias
- `Domain` no requiere conocer ningún concepto de persistencia ni orquestación.
- Los casos de uso en `Application` definen sus propias necesidades a través de sus puertos salientes.
