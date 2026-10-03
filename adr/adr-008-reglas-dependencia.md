# ADR-008: Reglas de Dependencias y Verificación Automatizada

## Estado
Aceptado

## Contexto
En arquitecturas en capas como Onion, es común que por descuido un controlador importe directamente un modelo de infraestructura o que el dominio use facades del framework, violando la regla de dependencias.

## Decisión
Se establecen 6 reglas de dependencia innegociables (R-01 a R-06):
- R-01: `Domain` no importa nada de terceros ni de Laravel (`grep -R "Illuminate\\" app/Domain` da 0).
- R-02: `Application` solo depende de `Domain`.
- R-03: `Presentation` jamás importa `Infrastructure`.
- R-04: Constructores de `Application` solo reciben tipos de `Domain` o puertos `Outbound`.
- R-05: Constructores de `Presentation` solo reciben puertos `Inbound`.
- R-06: Solo `Bootstrap` y proveedores enlazan clases de `Infrastructure`.

## Consecuencias
- La integridad arquitectónica es verificable mediante pruebas automáticas.
