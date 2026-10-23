# Estado de validación local

Revisión del entorno de desarrollo con Docker, PostgreSQL y Node disponibles.

## Ejecutado

- `docker compose config --quiet` en `test-manager-library-3239137-infra`: correcto.
- `docker compose exec -T api vendor/bin/phpunit`: correcto, 48 pruebas y 65 aserciones.
- `docker compose up -d --build api`: correcto; el arranque informó “Nothing to migrate”.
- `docker compose restart api`: correcto; `/api/v1/health` siguió respondiendo `{"status":"ok"}`.
- Base limpia temporal: la primera migración creó `public.migrations` y ejecutó las dos migraciones; la segunda respondió “Nothing to migrate”; `migrate:status` mostró ambas en estado `Ran`.
- `npm run build` en `test-manager-library-3239137-app`: correcto, 42 módulos.
- `node --check` para `seed.js`, `seed.test.js` y `catalogo.js`: correcto.
- `npm test` en `test-manager-library-3239137-tool`: correcto; la prueba simula la API, crea 15 préstamos en la primera pasada y confirma que la segunda no agrega más.
- Smoke contra la API real: `/api/v1/health` respondió `{"status":"ok"}`; el login de administrador devolvió `role=admin` y `expires_in=28800`; categorías, libros, socios, préstamos, vencidos y reporte respondieron 200.
- Contrato y autorización: sin token, 401; credenciales inválidas, 401 `INVALID_CREDENTIALS`; rol `librarian` en `POST /books` y en el reporte, 403; rango de fechas invertido, 422 `INVALID_DATE_RANGE`; libro inexistente, 404; rol `admin` solicitado por la API, 422.
- Base de datos: seis tablas de negocio, columnas y tipos correctos, claves foráneas con `ON DELETE RESTRICT`, `ck_book_available_range`, `ck_book_total_positive`, `ck_user_role`, índices y las cinco categorías de la semilla.
- Navegador: la pantalla de login carga en `http://localhost:5173`; no hubo errores de consola salvo el 404 de `favicon.ico`.
- `git diff --check`: sin errores de formato.

## Arranque repetible

El historial de migraciones se declara como `public.migrations` en `api/config/database.php`. Así Laravel no depende de que el esquema `library` exista para encontrar las migraciones ya ejecutadas.

Se comprobó de dos formas:

- Reinicio sobre la base existente: el contenedor volvió a arrancar y la API respondió.
- Base limpia temporal: la primera ejecución creó el historial y las tablas; la segunda no reejecutó nada.

El ajuste anterior de copiar filas a `library.migrations` fue local y ya no es necesario.

## Pendiente

- Pruebas de integración contra PostgreSQL real (T-12).
- Carrera con dos peticiones simultáneas por el último ejemplar (T-13 y RNF-2).
- Prueba de contrato HTTP completa sobre todas las rutas y códigos (T-16 y RNF-4).
- Prueba de carga con 5.000 libros y respuesta inferior a 300 ms (RNF-1).
- Pruebas dedicadas de autenticación, libros, reporte, hasher/token y portadas (T-07, T-08, T-11, T-14 y T-15).
- Recorrido autenticado completo de las siete pantallas y comprobación visual en móvil y escritorio (T-22 y T-23).
- Arranque completo desde un volumen limpio con creación del administrador y login (T-17).
- Sembrador contra la API real desde un volumen limpio (T-21).

## Limitaciones del entorno

- PHP y Composer no están instalados en el host; PHPUnit y las migraciones se ejecutan dentro del contenedor de la API.
- La validación del navegador cubrió la pantalla de login, no el flujo autenticado completo.
- La prueba del sembrador contra la API real no se repitió desde un volumen limpio; la base local ya tenía datos de corridas anteriores.
