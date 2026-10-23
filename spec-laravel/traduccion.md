# Traducción del spec a React + Laravel

El spec viene escrito dos veces, para Python y para .NET, y el reto se hace en
React y Laravel. Este documento es el puente: qué se respetó tal cual, qué
cambió de forma y qué decisiones hubo que tomar donde el spec no llegaba.

**No se editó ningún archivo de `spec-python/` ni de `spec-.net/`.** Son el
enunciado; esta carpeta es la respuesta.

---

## 1. Qué se respeta sin tocar

Todo lo que define el negocio, porque no depende del lenguaje:

- Las diez historias con sus criterios de aceptación (`spec.md` §3).
- Las seis reglas de negocio y los cinco requisitos no funcionales (§4 y §5).
- El contrato HTTP completo: rutas, códigos de estado, los catorce códigos de
  error, el sobre de paginación y el cuerpo de error único (`api-contract.md`).
- El modelo de datos: seis tablas, nombres de columna, tipos, nulabilidad,
  claves foráneas con su `ON DELETE`, índices y la semilla de categorías con sus
  UUID fijos (`data-model.md`).
- Los doce artículos de la constitución y las cuatro ADR.

`api-contract.md` y `data-model.md` son **idénticos** en las dos carpetas del
spec, lo que confirma que son la parte independiente de la tecnología. Se
tomaron como fuente única.

## 2. Equivalencias de tecnología

| # | Decisión | Spec (Python) | Aquí |
|---|---|---|---|
| D-01 | Lenguaje | Python 3.12 | PHP 8.3 |
| D-02 | Framework HTTP | FastAPI | Laravel 12 |
| D-03 | Persistencia | SQLAlchemy 2 + Alembic | Constructor de consultas de Laravel + migraciones |
| D-04 | Concurrencia | Testigo de versión + `CHECK` | Escritura condicionada sobre `available_copies` + el mismo `CHECK` (ver §3.1) |
| D-05 | Fechas y reloj | `timestamptz`, `Clock` como puerto | Igual: `timestampTz` y puerto `Clock` |
| D-06 | Reporte | Consulta agregada por un puerto de lectura | Igual: `MostBorrowedQuery` |
| D-07 | Objetos de valor | `dataclass(frozen=True)` | `final readonly class` |
| D-08 | Portadas | Disco o S3, clave opaca en la base | Igual, con el disco `covers` de Laravel |
| D-09 | Contraseñas | Argon2id tras un puerto | `password_hash`, con respaldo bcrypt (ver §4.2) |
| D-10 | Arranque | La aplicación crea el admin desde el entorno | Comando `library:ensure-admin` |
| D-11 | Token | JWT HS256, 8 horas | Igual, firmado con `hash_hmac` (ver §4.3) |
| D-12 | Pruebas | pytest con fakes | PHPUnit con dobles de prueba |

### Las capas

| Spec | Aquí | Qué no puede importar |
|---|---|---|
| `src/library/domain/` | `app/Domain/` | Nada salvo PHP |
| `src/library/application/` | `app/Application/` | Adaptadores ni framework |
| `src/library/adapters/` | `app/Adapters/` | — |
| `bootstrap/composition/port_bindings.py` | `app/Providers/PortBindingsProvider.php` | — |

### Las verificaciones de la constitución

| Artículo | El spec pide | Aquí |
|---|---|---|
| I, II | Contrato de `import-linter` y test en `tests/architecture/` | `tests/Architecture/LayerDependenciesTest.php`: lee los `use` de cada capa y falla si aparece uno prohibido. PHP no tiene `import-linter`, así que el contrato se escribe como prueba |
| III | Ninguna instanciación de adaptadores fuera de la composición | El mismo test busca `new <Adaptador>(` fuera de `app/Providers/` |
| V | `infra` no contiene `CREATE TABLE` | `grep -ri "create table" infra/` no devuelve nada |
| VII | Ninguna columna `status` ni `days_late` | No existen: el estado lo calcula `Loan::statusAt()` |
| VIII | Renombrar un libro no cambia el reporte | El reporte agrupa por los valores congelados de `loan_item` |
| IX | Ningún secreto en el repositorio | Solo `.env.example`; el `.env` está en `.gitignore` |
| XI | Código en inglés, mensajes en español | Clases, columnas y rutas en inglés; mensajes y comentarios en español |

## 3. Dónde el spec discrepaba consigo mismo

### 3.1 Falta la columna de versión

`plan.md` D-04 y [ADR-002](../spec-python/adr/adr-002-concurrencia-optimista.md)
piden un **testigo de versión por libro**. Pero `data-model.md` §3 lista las
columnas de `book` una por una y **no hay ninguna columna de versión**. Y el
artículo X es explícito: si el documento de datos discrepa, gana el documento de
datos.

Se respetó `data-model.md`: no se añadió la columna. El papel de testigo lo hace
`available_copies`, que ya estaba:

```sql
update library.book
   set available_copies = :nuevo
 where id = :id
   and available_copies = :lo_que_se_leyo   -- aquí está el testigo
   and deleted_at is null;
```

Si otra petición movió el contador entre la lectura y la escritura, la
actualización afecta cero filas y el servidor responde **409
`BOOK_NOT_AVAILABLE`**, que es lo que piden ADR-002 y RNF-2. El
`CHECK ck_book_available_range` sigue siendo la última barrera.

Es control optimista de verdad, no un bloqueo: no serializa la biblioteca y no
inventa una columna que el modelo de datos no declara.

### 3.2 T-18 cambia la clasificación de dos reglas

`data-model.md` §4 marca `ck_book_total_positive` y `ck_user_role` como
**pendiente (T-18)**, y T-18 dice que al hacerlo «`data-model.md` §4 se
actualiza». T-18 está hecho, así que esas dos filas pasan de **pendiente
(T-18)** a **motor**.

La corrección se anota aquí y no en `spec-python/data-model.md` porque el spec
original es el enunciado. Comprobación:

```sql
select conname, pg_get_constraintdef(oid)
  from pg_constraint
 where conname in ('ck_book_total_positive', 'ck_user_role');
```

## 4. Decisiones propias

### 4.1 Concurrencia

Ver §3.1.

### 4.2 Argon2id con respaldo

D-09 pide Argon2id. `password_hash` lo ofrece solo si el PHP se compiló con
soporte, lo que no está garantizado en toda imagen. El adaptador usa
`PASSWORD_ARGON2ID` cuando la constante existe y `PASSWORD_BCRYPT` si no. Lo que
no cambia en ningún caso: solo se guarda el hash, el dominio nunca ve la clave y
el hash no sale en ninguna respuesta.

### 4.3 El JWT, a mano

Son tres piezas en base64url y un HMAC de las dos primeras: unas treinta líneas.
La verificación compara con `hash_equals` para no filtrar la firma midiendo
tiempos. Añadir una dependencia para esto no se justificaba.

### 4.4 Los tres vencidos del sembrador

T-21 pide 3 líneas vencidas. El vencimiento es `loaned_at + 14 días` y lo calcula
el dominio; por la API no se puede pedir un préstamo con fecha vieja, y
`data-model.md` §9 prohíbe que el sembrador escriba SQL.

Se usó el asiento que el propio spec ofrece: **el reloj es un puerto** (D-05). La
API acepta la cabecera `X-Library-Clock-Offset-Days` **solo** si corre con
`DEMO_CLOCK=true` y `APP_ENV=local`. Fuera de local se ignora sin mirarla. Está
acotado a propósito y documentado en el README de `tool`.

### 4.5 Sin modelos de Eloquent

Se usó el constructor de consultas. Un modelo de Eloquent por tabla habría sido
un segundo dominio en paralelo, con sus propias reglas y su propia idea de qué es
un libro. El dominio ya existe en `app/Domain/`; los repositorios solo traducen
filas a esos objetos y al revés.

### 4.6 Laravel y el esquema `library`

Tres detalles que Laravel no da por defecto:

- Las tablas se nombran `library.book` y la conexión lleva
  `search_path=library,public`. Se nombra el esquema explícito para que una
  conexión mal configurada falle en vez de escribir en `public`.
- `user` es palabra reservada; el constructor de consultas la entrecomilla solo,
  y en SQL a mano se escribe `library."user"`.
- Las claves son UUID de la aplicación y **no hay** `created_at` ni
  `updated_at`: ningún requisito los pide (data-model §8).

## 5. Estado de las tareas

| Tarea | Estado | Evidencia |
|---|---|---|
| T-01 Esqueleto y `/health` | implementado; smoke local correcto | `GET /api/v1/health` respondió `{"status":"ok"}` |
| T-02 Migración inicial con semilla | implementado; esquema y arranque repetible verificados | Seis tablas de negocio y cinco categorías; historial en `public.migrations`; doble migración sobre base limpia sin cambios |
| T-03 Objetos de valor | hecho | `tests/Unit/Domain/ValueObjectsTest.php` |
| T-04 `Book` | hecho | `tests/Unit/Domain/BookTest.php` |
| T-05 `Loan` | hecho | `LoanTest.php`, `MemberBorrowingLimitTest.php` |
| T-06 `Member` y `User` | hecho | `UserTest.php` |
| T-07 Autenticación | hecho | `LoginService`, `CreateLibrarianService` |
| T-08 Libros | hecho | Siete casos de uso en `UseCase/Book` |
| T-09 Prestar | hecho | `LendBooksServiceTest.php`, con la carrera por el último ejemplar |
| T-10 Devolver y vencidos | hecho | `ReturnLoanItemServiceTest.php` |
| T-11 Reporte | hecho | `SqlMostBorrowedQuery` agrupa por valores congelados |
| T-12 Repositorios | implementado; faltan pruebas de integración PostgreSQL | `app/Adapters/Persistence/Database/` |
| T-13 Préstamos y testigo de versión | implementado; falta carrera entre peticiones reales | Escritura condicionada; prueba actual con doble; ver §3.1 |
| T-14 Hash y token | implementado; el emisor de token tiene prueba; falta la prueba del hasher | `NativePasswordHasher`, `JwtTokenIssuer`, `JwtTokenIssuerTest` |
| T-15 Portadas | implementado; falta prueba integrada de subida | `DiskCoverStorage`, tipo validado por contenido |
| T-16 Rutas y cuerpo de error | implementado; falta prueba de contrato HTTP completa | `routes/api.php`, `ErrorResponder` |
| T-17 Composición y arranque | implementado; login admin local correcto; falta arranque desde base vacía | `PortBindingsProvider`, `library:ensure-admin` |
| T-18 Invariantes al motor | hecho | Segunda migración; ver §3.2 |
| T-19 Pruebas de arquitectura | hecho | `LayerDependenciesTest.php` |
| T-20 `infra` con un comando | implementado; compose y reinicio verificados | `docker-compose.yml`; tres servicios |
| T-21 Sembrador por la API | implementado; idempotencia simulada e ISBN-13 validado para los 40 libros; falta probar contra API real | `tool/seed.js`, `tool/catalogo.js` y `tool/seed.test.js`; los 3 vencidos, ver §4.4 |
| T-22 `app` | implementado; build de producción comprobado; falta recorrido manual integrado | Siete pantallas |
| T-23 `page` | implementado; falta comprobación visual en móvil y escritorio | Sitio estático |

**Validación pendiente:** pruebas de integración contra PostgreSQL real y la
prueba concurrente con dos peticiones simultáneas; RNF-2 se cubre con un doble
que reproduce la lectura vieja, no con dos procesos. También falta la prueba de
carga de RNF-1 con 5 000 libros: el sembrador deja 40. El estado de ejecución y
las limitaciones del entorno están en [`estado-validacion.md`](estado-validacion.md).
