# Estado de validación local

Revisión del entorno de desarrollo con Docker, PostgreSQL y Node disponibles.

## Ejecutado

- `docker compose config --quiet` en `test-manager-library-3239137-infra`: correcto.
- `docker compose exec -T api vendor/bin/phpunit`: correcto, 55 pruebas y 73 aserciones.
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

## Validaciones completadas en CI

- API contra PostgreSQL real: contrato HTTP, autenticación, permisos, libros, arranque idempotente y hasher.
- Concurrencia: dos procesos compiten por el último ejemplar y solo uno obtiene el préstamo.
- Rendimiento: búsqueda con 5.000 libros dentro del límite de 300 ms.
- Frontend: instalación reproducible y compilación de producción.
- Sitio público y documentación: validaciones automatizadas de sus workflows.
- Infraestructura: configuración de Compose e integración completa con base limpia; el recorrido autenticado de escritorio y la navegación móvil en las siete pantallas pasan en Playwright.
- Sembrador: cuatro pruebas automatizadas, incluida la validación ISBN-13 y la idempotencia.

## Limitaciones del entorno

- PHP y Composer no están instalados en el host; PHPUnit y las migraciones se ejecutan dentro del contenedor de la API.
- El recorrido completo de navegador y las pruebas PostgreSQL se ejecutaron en GitHub Actions, no en el host local.
- `npm run build` no pudo ejecutarse localmente porque Vite no estaba instalado en `node_modules`; la compilación sí fue validada por el workflow del frontend.

## Verificacion local actual

Pruebas Node en tool: 4 aprobadas. Se valido el checksum ISBN-13 de los 40 libros y se corrigio el generador. Tambien se comprobo que el sembrador rechaza una ejecucion sin ADMIN_PASSWORD y que dos corridas simuladas no duplican prestamos. El entorno actual no tiene Docker, PHP ni Composer; por eso las pruebas de PostgreSQL, Laravel, interfaz integrada y carga no se repitieron. La evidencia anterior de este documento es historica y corresponde a ejecuciones previas.

## Evidencia y controles de repositorio

Se agregaron pruebas de integracion para PostgreSQL que cubren el contrato HTTP de salud, autenticacion, permisos, alta/consulta/baja de libros, rango de fechas y el arranque idempotente del administrador. Tambien se agrego una carrera con dos procesos PostgreSQL sobre el ultimo ejemplar y una prueba de busqueda del catalogo con 5.000 libros y el limite de 300 ms. La suite se ejecuta con `composer test` y requiere PHP 8.3, `pdo_pgsql`, PostgreSQL y `pcntl` para la prueba concurrente.

En el host local faltan Docker, PHP y Composer, y Vite no está instalado en `node_modules`; por eso esas validaciones locales no están disponibles. Esta limitación no afecta a los resultados de CI, donde las comprobaciones de los seis repositorios terminaron correctamente.

Las reglas de protección están activas en GitHub para `main` en los seis repositorios y para `develop` en los repositorios de código. Requieren PR y el workflow correspondiente; también bloquean el push directo y conservan el historial lineal.
