# ADR-010: Contenedorización y Entornos Aislados con Docker Compose

## Estado
Aceptado

## Contexto
El entorno de evaluación debe ser reproducible, portable e independiente del sistema operativo del evaluador o del aprendiz. No debe asumirse la presencia de PHP, Composer, Node o bases de datos instaladas en la máquina anfitriona.

## Decisión
- Toda la infraestructura se orquesta mediante `docker-compose.yml` en el repositorio `test-simple-stock-flow-infra`.
- Contenedor de base de datos relacional con motor limpio.
- Contenedor de API Laravel con PHP 8.2 FPM / Artisan Server.
- Contenedor de Frontend React servido mediante Nginx con proxy inverso hacia `/api` y `/media`.
- Manejo de dependencias (`vendor`, `node_modules`) mediante volúmenes con nombre para evitar problemas de permisos cruzados en Windows/Linux.

## Consecuencias
- Despliegue reproducible con un solo comando: `docker compose up -d`.
