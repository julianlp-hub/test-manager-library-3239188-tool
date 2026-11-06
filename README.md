# test-manager-library-3239188-tool

Utilidades de *Manager Library*. Contiene el **sembrador de datos de demostración**.
Este repositorio entrega el **diseño del sembrador** guiado por el spec (T-21).

## Principio clave

El sembrador usa **la API, nunca SQL directo** (data-model.md §9). Así los datos pasan
por las mismas reglas de negocio que los datos reales: límites, vencimientos,
congelado de título y categoría, y disponibles.

## Requisitos previos

- La API arriba (`GET /health` responde `{"status":"ok"}`).
- La base con las migraciones aplicadas (las 5 categorías ya sembradas por la migración).
- El administrador inicial creado por el arranque con las variables de entorno.

## Variables de entorno

| Variable | Ejemplo | Para qué |
|---|---|---|
| API_URL | http://localhost:8000/api/v1 | Base de la API |
| ADMIN_USERNAME | admin | Usuario admin inicial |
| ADMIN_PASSWORD | (desde entorno) | Nunca se escribe en el repo |

Se versiona solo `.env.example`; el `.env` real está en `.gitignore` (Constitución, art. IX).

## Meta de datos (evidencia de T-21)

| Dato | Cantidad |
|---|---|
| Libros | al menos 40, repartidos en las 5 categorías |
| Socios | 10 |
| Préstamos | 15 |
| Préstamos con líneas vencidas | 3 |

Tras correrlo: `GET /loans/overdue` devuelve 3 líneas y el reporte tiene filas.

## Orden de llamadas

1. `POST /auth/login` como admin → token.
2. `GET /categories` → ids de las 5 categorías.
3. `POST /users` → 2 bibliotecarios de demostración.
4. `POST /books` × 40 o más (ISBN de 13 dígitos únicos, total de ejemplares entre 1 y 5).
5. `POST /members` × 10 (documentos únicos).
6. Iniciar sesión como bibliotecario y `POST /loans` × 15 (1 a 3 libros sin repetir, máx. 3 sin devolver por socio).
7. Devolver algunas líneas con `POST /loans/{id}/items/{id}/return`.
8. Marcar 3 préstamos como vencidos (ver abajo).

## Cómo lograr los 3 vencidos

Un préstamo vence a los 14 días, y la API no permite fechas en el pasado. Opciones:

- **Reloj controlable:** la API usa el puerto `Clock`. En el entorno de demostración se inyecta un reloj
  fijado 15 días atrás solo para esos 3 préstamos y luego se restaura.
- **Registrar antes de avanzar el reloj:** crear los 3 préstamos con el reloj retrasado.

No se edita la base a mano: sería saltarse las reglas del sistema.

## Idempotencia y reintentos

- Si un ISBN o un documento ya existe, la API responde 409 (`ISBN_TAKEN`, `DOCUMENT_TAKEN`):
  el sembrador lo trata como "ya sembrado" y continúa.
- Se puede correr varias veces sin duplicar datos.

## Cómo se ejecutaría

```
cp .env.example .env
# editar API_URL y credenciales
<comando del sembrador>
```

## Usos adicionales

- **Prueba de carga (RNF-1):** con parámetro `--books 5000` genera 5 000 libros para medir que
  la búsqueda responda en menos de 300 ms.
- **Prueba de concurrencia (RNF-2):** dos sesiones piden el último ejemplar a la vez; una recibe 201 y la otra 409.

## Trazabilidad

| Tarea | Evidencia |
|---|---|
| T-21 | Tras correrlo, `/loans/overdue` devuelve 3 líneas y el reporte tiene filas |
| RNF-1 | Búsqueda < 300 ms con 5 000 libros |
| RNF-2 | Último ejemplar: uno gana, otro 409 |

## Estado

Diseño documentado.