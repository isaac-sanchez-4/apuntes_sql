# 🗄️ Guía completa de SQL

> **Objetivo:** aprender SQL desde cero hasta consultas avanzadas, diseño de bases de datos y optimización. Los ejemplos usan una sintaxis cercana a PostgreSQL/SQL estándar. Algunas instrucciones (`LIMIT`, `TOP`, `AUTO_INCREMENT`, tipos de datos, procedimientos, etc.) cambian entre PostgreSQL, MySQL, SQL Server, Oracle y SQLite.

---

## 0. ¿Qué es SQL?

**SQL** (*Structured Query Language*) es el lenguaje utilizado para trabajar con bases de datos relacionales. Permite:

- consultar datos;
- insertar, modificar y borrar registros;
- crear y modificar tablas, índices y vistas;
- controlar transacciones;
- gestionar usuarios y permisos.

Una base de datos relacional organiza la información en **tablas**. Cada tabla tiene:

- **filas** o registros: elementos concretos;
- **columnas** o campos: propiedades de esos elementos;
- **claves**: identificadores y relaciones entre tablas.

### Motor, servidor y esquema

- **SGBD/DBMS:** programa que administra la base de datos: PostgreSQL, MySQL, SQL Server, Oracle, SQLite…
- **base de datos:** conjunto de objetos y datos.
- **esquema:** espacio lógico que agrupa tablas, vistas, funciones, etc.
- **tabla:** estructura formada por columnas y filas.

SQL no es, en general, un lenguaje sensible a mayúsculas en palabras reservadas: `SELECT` y `select` suelen significar lo mismo. Sí importa el uso de comillas y, normalmente, el nombre exacto de identificadores sensibles a mayúsculas.

---

# 1. Categorías de instrucciones SQL

## DQL: consultar datos

```sql
SELECT * FROM empleados;
```

## DDL: definir estructuras

```sql
CREATE TABLE departamentos (...);
ALTER TABLE empleados ADD COLUMN email VARCHAR(255);
DROP TABLE empleados;
TRUNCATE TABLE empleados;
```

## DML: manipular datos

```sql
INSERT INTO empleados (...) VALUES (...);
UPDATE empleados SET salario = 2000 WHERE id = 1;
DELETE FROM empleados WHERE id = 1;
```

`MERGE` y `UPSERT` también se utilizan para insertar o actualizar según exista el registro, aunque su sintaxis depende del motor.

## DCL: permisos

```sql
GRANT SELECT ON empleados TO analista;
REVOKE INSERT ON empleados FROM analista;
```

## TCL: transacciones

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
COMMIT;
-- o ROLLBACK; si algo falla
```

---

# 2. Crear una base de datos y tablas

La sintaxis exacta depende del SGBD:

```sql
CREATE DATABASE empresa;
```

Una tabla de ejemplo completa:

```sql
CREATE TABLE departamentos (
    id              INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL UNIQUE,
    presupuesto     NUMERIC(12, 2) NOT NULL DEFAULT 0,
    creado_en       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT presupuesto_no_negativo CHECK (presupuesto >= 0)
);

CREATE TABLE empleados (
    id              INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,
    email           VARCHAR(255) NOT NULL UNIQUE,
    salario         NUMERIC(10, 2) NOT NULL CHECK (salario > 0),
    fecha_ingreso   DATE NOT NULL DEFAULT CURRENT_DATE,
    activo          BOOLEAN NOT NULL DEFAULT TRUE,
    departamento_id INTEGER,
    jefe_id         INTEGER,

    CONSTRAINT fk_empleado_departamento
        FOREIGN KEY (departamento_id)
        REFERENCES departamentos(id)
        ON UPDATE CASCADE
        ON DELETE SET NULL,

    CONSTRAINT fk_empleado_jefe
        FOREIGN KEY (jefe_id)
        REFERENCES empleados(id)
        ON DELETE SET NULL
);
```

## Tipos de datos habituales

| Categoría | Tipos comunes | Uso |
|---|---|---|
| Enteros | `SMALLINT`, `INTEGER`, `BIGINT` | cantidades e identificadores |
| Decimales | `DECIMAL(p,s)`, `NUMERIC(p,s)` | dinero y cálculos exactos |
| Aproximados | `REAL`, `FLOAT`, `DOUBLE PRECISION` | mediciones donde se acepta aproximación |
| Texto | `CHAR(n)`, `VARCHAR(n)`, `TEXT` | cadenas |
| Fecha/hora | `DATE`, `TIME`, `TIMESTAMP` | fechas y momentos |
| Booleano | `BOOLEAN` | verdadero/falso |
| Binario | `BLOB`, `BYTEA`, `VARBINARY` | archivos o bytes |
| Estructurado | `JSON`, `JSONB`, `XML`, arrays | datos semiestructurados, según el motor |

Para importes monetarios es preferible `NUMERIC/DECIMAL`, no `FLOAT`, porque los tipos de coma flotante pueden introducir errores de redondeo.

## Restricciones (*constraints*)

- `PRIMARY KEY`: identifica unívocamente cada fila; implica `NOT NULL`.
- `FOREIGN KEY`: obliga a que un valor exista en otra tabla.
- `UNIQUE`: impide duplicados.
- `NOT NULL`: obliga a proporcionar un valor.
- `CHECK`: valida una condición.
- `DEFAULT`: valor usado cuando no se proporciona una columna.

Una clave primaria compuesta:

```sql
CREATE TABLE matriculas (
    alumno_id INTEGER NOT NULL,
    curso_id  INTEGER NOT NULL,
    nota      NUMERIC(4, 2),
    PRIMARY KEY (alumno_id, curso_id),
    CHECK (nota IS NULL OR nota BETWEEN 0 AND 10)
);
```

---

# 3. Modificar estructuras

```sql
ALTER TABLE empleados ADD COLUMN telefono VARCHAR(30);
ALTER TABLE empleados ALTER COLUMN telefono SET DEFAULT 'No indicado';
ALTER TABLE empleados DROP COLUMN telefono;
ALTER TABLE empleados RENAME COLUMN nombre TO nombre_completo;
ALTER TABLE empleados RENAME TO trabajadores;
```

Añadir o quitar restricciones:

```sql
ALTER TABLE empleados
    ADD CONSTRAINT salario_positivo CHECK (salario > 0);

ALTER TABLE empleados
    DROP CONSTRAINT salario_positivo;
```

`DROP TABLE` elimina la tabla y sus datos. `TRUNCATE` vacía la tabla rápidamente, pero su comportamiento con transacciones, claves foráneas y reinicio de identidades depende del motor. Usa siempre `WHERE` en un `DELETE` salvo que quieras borrar todas las filas.

---

# 4. INSERT: insertar datos

```sql
INSERT INTO departamentos (nombre, presupuesto)
VALUES ('Ventas', 100000), ('Informática', 250000);

INSERT INTO empleados
    (nombre, email, salario, departamento_id)
VALUES
    ('Ana García', 'ana@empresa.com', 2200, 1),
    ('Luis Pérez', 'luis@empresa.com', 2800, 2);
```

Insertar a partir de una consulta:

```sql
INSERT INTO empleados_historico (id, nombre, salario)
SELECT id, nombre, salario
FROM empleados
WHERE activo = FALSE;
```

En algunos motores se puede recuperar la fila insertada con `RETURNING` o `OUTPUT`:

```sql
INSERT INTO departamentos (nombre)
VALUES ('Marketing')
RETURNING id, nombre;
```

Evita insertar sin indicar las columnas:

```sql
-- Frágil: depende del orden de la tabla
INSERT INTO empleados VALUES (...);
```

---

# 5. SELECT: consultar datos

```sql
SELECT nombre, salario
FROM empleados;

SELECT DISTINCT departamento_id
FROM empleados;

SELECT *
FROM empleados;
```

`SELECT *` es útil para explorar, pero en producción conviene indicar las columnas: evita traer datos innecesarios y hace más estable el contrato de la consulta.

## Alias

```sql
SELECT
    e.nombre AS empleado,
    e.salario AS sueldo_mensual
FROM empleados AS e;
```

## Expresiones y `CASE`

```sql
SELECT
    nombre,
    salario,
    salario * 12 AS salario_anual,
    CASE
        WHEN salario >= 3000 THEN 'Alto'
        WHEN salario >= 1800 THEN 'Medio'
        ELSE 'Bajo'
    END AS nivel_salarial
FROM empleados;
```

## Orden de evaluación lógico

Aunque se escribe en otro orden, una consulta se interpreta aproximadamente así:

1. `FROM` y `JOIN`
2. `WHERE`
3. `GROUP BY`
4. `HAVING`
5. funciones de ventana
6. `SELECT`
7. `DISTINCT`
8. `ORDER BY`
9. `LIMIT`/`OFFSET` o `FETCH`

Por eso no se puede usar normalmente un alias de `SELECT` dentro de `WHERE`, y una función de ventana no se puede filtrar directamente en `WHERE`.

---

# 6. Filtrar: WHERE

```sql
SELECT nombre, salario
FROM empleados
WHERE activo = TRUE
  AND salario BETWEEN 1800 AND 3000
  AND nombre LIKE 'A%';
```

Operadores importantes:

```sql
WHERE salario > 2000
WHERE departamento_id IN (1, 2, 3)
WHERE nombre NOT LIKE '%test%'
WHERE fecha_ingreso >= DATE '2024-01-01'
WHERE salario IS NULL
WHERE salario IS NOT NULL
```

## `NULL`: ausencia de valor

`NULL` no es cero, cadena vacía ni `FALSE`. Representa un valor desconocido o inexistente. Las comparaciones con `NULL` no devuelven `TRUE`, por lo que esto es incorrecto:

```sql
-- Incorrecto
WHERE departamento_id = NULL
```

Debe escribirse:

```sql
WHERE departamento_id IS NULL;
```

SQL usa lógica de tres valores: `TRUE`, `FALSE` y `UNKNOWN`. Comprueba siempre cómo afectan los nulos a tus filtros.

Funciones útiles:

```sql
SELECT
    COALESCE(telefono, 'Sin teléfono') AS telefono,
    NULLIF(codigo, '') AS codigo_nulo
FROM clientes;
```

`COALESCE(a, b, c)` devuelve el primer valor no nulo. `NULLIF(a, b)` devuelve `NULL` si `a = b`.

---

# 7. Ordenar y limitar resultados

```sql
SELECT nombre, salario
FROM empleados
ORDER BY salario DESC, nombre ASC;
```

- `ASC`: ascendente, valor por defecto.
- `DESC`: descendente.
- Los `NULL` pueden ordenarse como `NULLS FIRST` o `NULLS LAST`, según soporte del motor.

```sql
SELECT *
FROM empleados
ORDER BY id
LIMIT 10 OFFSET 20;
```

Para paginación grande suele ser mejor la **paginación por cursor**:

```sql
SELECT id, nombre
FROM empleados
WHERE id > 1000
ORDER BY id
LIMIT 50;
```

Usa siempre un orden determinista, incluyendo una columna única si hay empates.

---

# 8. Funciones integradas

## Texto

```sql
SELECT
    UPPER(nombre) AS nombre_mayusculas,
    LOWER(email) AS email_minusculas,
    TRIM(nombre) AS nombre_limpio,
    LENGTH(nombre) AS longitud,
    CONCAT(nombre, ' <', email, '>') AS contacto
FROM empleados;
```

La concatenación exacta varía (`||`, `CONCAT`, `+`). Otras funciones habituales: `SUBSTRING`, `REPLACE`, `POSITION`, `LPAD`, `RPAD`, `LEFT`, `RIGHT`.

## Números

```sql
SELECT
    ROUND(salario, 0),
    CEIL(salario),
    FLOOR(salario),
    ABS(salario)
FROM empleados;
```

## Fechas

```sql
SELECT
    CURRENT_DATE,
    CURRENT_TIMESTAMP,
    EXTRACT(YEAR FROM fecha_ingreso) AS anio,
    fecha_ingreso + INTERVAL '30 days' AS revision
FROM empleados;
```

`DATE_ADD`, `DATEDIFF`, `DATE_PART` y la sintaxis de intervalos cambian entre motores.

---

# 9. Agregaciones: COUNT, SUM, AVG, MIN y MAX

Las funciones de agregación convierten muchas filas en un resultado por grupo:

```sql
SELECT
    departamento_id,
    COUNT(*) AS cantidad,
    COUNT(email) AS emails_no_nulos,
    SUM(salario) AS masa_salarial,
    AVG(salario) AS salario_medio,
    MIN(salario) AS salario_minimo,
    MAX(salario) AS salario_maximo
FROM empleados
GROUP BY departamento_id;
```

`COUNT(*)` cuenta filas; `COUNT(columna)` ignora los `NULL`; `SUM` y `AVG` también ignoran los `NULL`.

## HAVING

`WHERE` filtra filas antes de agrupar. `HAVING` filtra grupos después de agregar:

```sql
SELECT departamento_id, AVG(salario) AS media
FROM empleados
WHERE activo = TRUE
GROUP BY departamento_id
HAVING AVG(salario) > 2500;
```

Toda columna seleccionada que no sea agregada debe aparecer normalmente en `GROUP BY`.

---

# 10. JOINs: combinar tablas

## INNER JOIN
Solo devuelve coincidencias en ambas tablas:

```sql
SELECT e.nombre, d.nombre AS departamento
FROM empleados AS e
INNER JOIN departamentos AS d
    ON d.id = e.departamento_id;
```

## LEFT JOIN
Devuelve todas las filas de la izquierda y las coincidencias de la derecha. Si no hay coincidencia, las columnas derechas son `NULL`:

```sql
SELECT d.nombre, e.nombre
FROM departamentos AS d
LEFT JOIN empleados AS e
    ON e.departamento_id = d.id;
```

Encontrar departamentos sin empleados:

```sql
SELECT d.*
FROM departamentos AS d
LEFT JOIN empleados AS e ON e.departamento_id = d.id
WHERE e.id IS NULL;
```

## RIGHT JOIN y FULL OUTER JOIN

- `RIGHT JOIN`: conserva todas las filas de la tabla derecha.
- `FULL OUTER JOIN`: conserva las filas de ambas tablas, coincidan o no.

Se pueden reescribir muchos `RIGHT JOIN` cambiando el orden de las tablas para mejorar la legibilidad. SQLite no soporta todos los tipos de `JOIN` de la misma forma.

## CROSS JOIN
Producto cartesiano: combina cada fila con todas las de la otra tabla.

```sql
SELECT colores.nombre, tallas.nombre
FROM colores
CROSS JOIN tallas;
```

## SELF JOIN
Una tabla se une consigo misma:

```sql
SELECT
    empleado.nombre AS empleado,
    jefe.nombre AS jefe
FROM empleados AS empleado
LEFT JOIN empleados AS jefe
    ON jefe.id = empleado.jefe_id;
```

### Cuidado con los filtros de un LEFT JOIN

Este filtro convierte de hecho el `LEFT JOIN` en un `INNER JOIN`:

```sql
-- Puede eliminar departamentos sin empleados de 2025
FROM departamentos d
LEFT JOIN empleados e ON e.departamento_id = d.id
WHERE e.fecha_ingreso >= DATE '2025-01-01'
```

Si quieres conservarlos, filtra en `ON`:

```sql
FROM departamentos d
LEFT JOIN empleados e
  ON e.departamento_id = d.id
 AND e.fecha_ingreso >= DATE '2025-01-01'
```

---

# 11. Subconsultas

## En WHERE

```sql
SELECT nombre, salario
FROM empleados
WHERE salario > (
    SELECT AVG(salario)
    FROM empleados
);
```

## En SELECT

```sql
SELECT
    nombre,
    salario,
    (SELECT MAX(salario) FROM empleados) AS salario_maximo
FROM empleados;
```

## En FROM: tabla derivada

```sql
SELECT departamento_id, media
FROM (
    SELECT departamento_id, AVG(salario) AS media
    FROM empleados
    GROUP BY departamento_id
) AS resumen
WHERE media > 2500;
```

## Subconsulta correlacionada
Se ejecuta conceptualmente para cada fila exterior:

```sql
SELECT e.nombre, e.salario
FROM empleados e
WHERE e.salario > (
    SELECT AVG(e2.salario)
    FROM empleados e2
    WHERE e2.departamento_id = e.departamento_id
);
```

## EXISTS y NOT EXISTS
Para comprobar existencia, normalmente es preferible `EXISTS` a contar filas:

```sql
SELECT d.nombre
FROM departamentos d
WHERE EXISTS (
    SELECT 1
    FROM empleados e
    WHERE e.departamento_id = d.id
);
```

## IN y NOT IN: cuidado con NULL

`NOT IN` puede producir resultados inesperados si la subconsulta contiene `NULL`. En consultas de ausencia suele ser más seguro:

```sql
SELECT d.*
FROM departamentos d
WHERE NOT EXISTS (
    SELECT 1
    FROM empleados e
    WHERE e.departamento_id = d.id
);
```

---

# 12. CTEs (`WITH`)

Una CTE da nombre a un resultado intermedio y mejora la legibilidad:

```sql
WITH empleados_top AS (
    SELECT nombre, departamento_id, salario
    FROM empleados
    WHERE salario > 3000
)
SELECT nombre, departamento_id
FROM empleados_top
WHERE departamento_id = 2;
```

Varias CTEs:

```sql
WITH ventas_por_cliente AS (
    SELECT cliente_id, SUM(total) AS total
    FROM ventas
    GROUP BY cliente_id
), clientes_frecuentes AS (
    SELECT cliente_id, total
    FROM ventas_por_cliente
    WHERE total > 10000
)
SELECT c.nombre, f.total
FROM clientes c
JOIN clientes_frecuentes f ON f.cliente_id = c.id;
```

Una CTE no es necesariamente una tabla temporal física ni siempre mejora el rendimiento; depende del optimizador y del motor.

## CTE recursiva
Para jerarquías, árboles o secuencias:

```sql
WITH RECURSIVE organigrama AS (
    SELECT id, nombre, jefe_id, 0 AS nivel
    FROM empleados
    WHERE jefe_id IS NULL

    UNION ALL

    SELECT e.id, e.nombre, e.jefe_id, o.nivel + 1
    FROM empleados e
    JOIN organigrama o ON e.jefe_id = o.id
)
SELECT *
FROM organigrama
ORDER BY nivel, nombre;
```

Toda recursión necesita una parte inicial y una parte recursiva; diseña una condición que evite ciclos infinitos.

---

# 13. Funciones de ventana

Calculan sobre un conjunto relacionado de filas sin eliminar el detalle de cada fila.

## Sintaxis

```sql
funcion(...) OVER (
    PARTITION BY columna
    ORDER BY columna
    ROWS BETWEEN ... AND ...
)
```

## Media por departamento

```sql
SELECT
    nombre,
    departamento_id,
    salario,
    AVG(salario) OVER (PARTITION BY departamento_id) AS media_departamento
FROM empleados;
```

## Ranking

```sql
SELECT
    nombre,
    departamento_id,
    salario,
    ROW_NUMBER() OVER (
        PARTITION BY departamento_id ORDER BY salario DESC, id
    ) AS numero,
    RANK() OVER (
        PARTITION BY departamento_id ORDER BY salario DESC
    ) AS ranking,
    DENSE_RANK() OVER (
        PARTITION BY departamento_id ORDER BY salario DESC
    ) AS ranking_denso
FROM empleados;
```

- `ROW_NUMBER`: siempre numera 1, 2, 3…; el desempate debe ser determinista.
- `RANK`: deja huecos tras empates: 1, 2, 2, 4.
- `DENSE_RANK`: no deja huecos: 1, 2, 2, 3.
- `NTILE(n)`: divide las filas en `n` grupos.

## Acumulados y marcos

```sql
SELECT
    fecha,
    importe,
    SUM(importe) OVER (
        ORDER BY fecha
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS acumulado,
    AVG(importe) OVER (
        ORDER BY fecha
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS media_movil_7_filas
FROM ventas;
```

`ROWS`, `RANGE` y `GROUPS` no se comportan igual ante empates. Especifica el marco cuando el resultado deba ser exacto.

## LAG, LEAD, FIRST_VALUE y LAST_VALUE

```sql
SELECT
    fecha,
    importe,
    LAG(importe) OVER (ORDER BY fecha) AS importe_anterior,
    LEAD(importe) OVER (ORDER BY fecha) AS importe_siguiente,
    importe - LAG(importe) OVER (ORDER BY fecha) AS variacion
FROM ventas;
```

Para filtrar por el resultado de una ventana, usa una subconsulta o CTE:

```sql
WITH clasificados AS (
    SELECT e.*, ROW_NUMBER() OVER (
        PARTITION BY departamento_id ORDER BY salario DESC, id
    ) AS puesto
    FROM empleados e
)
SELECT *
FROM clasificados
WHERE puesto <= 3;
```

---

# 14. UNION, INTERSECT y EXCEPT

Combinan resultados compatibles: mismo número de columnas y tipos compatibles.

```sql
SELECT email FROM clientes
UNION
SELECT email FROM proveedores;
```

- `UNION`: elimina duplicados.
- `UNION ALL`: conserva duplicados y suele ser más rápido.
- `INTERSECT`: elementos presentes en ambos resultados.
- `EXCEPT`: elementos del primer resultado que no están en el segundo.

El `ORDER BY` se coloca al final de la combinación completa, salvo que se usen subconsultas.

---

# 15. UPDATE y DELETE

```sql
UPDATE empleados
SET salario = salario * 1.05,
    actualizado_en = CURRENT_TIMESTAMP
WHERE departamento_id = 2;
```

Comprueba primero el `WHERE` con un `SELECT` equivalente y usa una transacción para cambios importantes:

```sql
BEGIN;
UPDATE empleados
SET activo = FALSE
WHERE fecha_ingreso < DATE '2018-01-01';

-- Verifica antes de confirmar
SELECT COUNT(*) FROM empleados WHERE activo = FALSE;
COMMIT;
-- ROLLBACK; si el resultado no es correcto
```

```sql
DELETE FROM empleados
WHERE activo = FALSE;
```

Una alternativa más segura que `DELETE` en muchos sistemas es el **borrado lógico**, por ejemplo `activo = FALSE` o `eliminado_en`, cuando se necesita conservar el historial.

## UPSERT / MERGE

La implementación depende del motor. En PostgreSQL:

```sql
INSERT INTO departamentos (nombre, presupuesto)
VALUES ('Ventas', 120000)
ON CONFLICT (nombre)
DO UPDATE SET presupuesto = EXCLUDED.presupuesto;
```

`MERGE` permite expresar inserciones, actualizaciones y borrados condicionales en una sola operación en motores que lo soportan.

---

# 16. Transacciones y concurrencia

Una transacción agrupa operaciones que deben confirmarse o deshacerse juntas. Propiedades **ACID**:

- **Atomicidad:** todo o nada.
- **Consistencia:** se respetan reglas y restricciones.
- **Aislamiento:** las transacciones concurrentes no se interfieren incorrectamente.
- **Durabilidad:** un `COMMIT` confirmado sobrevive a fallos normales.

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE id = 1;
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 2;
COMMIT;
```

Si falla cualquier paso:

```sql
ROLLBACK;
```

Puntos de guardado:

```sql
SAVEPOINT antes_de_bono;
ROLLBACK TO SAVEPOINT antes_de_bono;
RELEASE SAVEPOINT antes_de_bono;
```

## Niveles de aislamiento

De menor a mayor aislamiento, los nombres habituales son:

- `READ UNCOMMITTED`;
- `READ COMMITTED`;
- `REPEATABLE READ`;
- `SERIALIZABLE`.

Pueden controlar lecturas sucias, lecturas no repetibles y filas fantasma. El comportamiento exacto depende del motor. Los bloqueos explícitos también son específicos:

```sql
SELECT *
FROM cuentas
WHERE id = 1
FOR UPDATE;
```

Mantén las transacciones cortas y en un orden consistente para reducir bloqueos y deadlocks.

---

# 17. Índices y rendimiento

Un índice acelera búsquedas, joins, ordenaciones y restricciones `UNIQUE`, pero ocupa espacio y hace más costosos `INSERT`, `UPDATE` y `DELETE`.

```sql
CREATE INDEX idx_empleados_departamento
    ON empleados (departamento_id);

CREATE INDEX idx_empleados_departamento_salario
    ON empleados (departamento_id, salario DESC);

CREATE UNIQUE INDEX idx_empleados_email
    ON empleados (email);
```

## Reglas prácticas

1. Indexa columnas usadas frecuentemente en `WHERE`, `JOIN`, `ORDER BY` y claves foráneas.
2. En índices compuestos, el orden de columnas importa: suele ir primero la columna más útil para los filtros.
3. No indexes todas las columnas sin medir.
4. Evita aplicar funciones a una columna indexada si impiden usar el índice:

```sql
-- Puede impedir un índice normal sobre fecha_ingreso
WHERE EXTRACT(YEAR FROM fecha_ingreso) = 2025
```

Mejor usa un intervalo:

```sql
WHERE fecha_ingreso >= DATE '2025-01-01'
  AND fecha_ingreso <  DATE '2026-01-01'
```

5. No uses `SELECT *` si solo necesitas pocas columnas.
6. Evita traer millones de filas para filtrarlas en la aplicación.
7. Usa consultas parametrizadas, no concatenes entrada del usuario.

## EXPLAIN

```sql
EXPLAIN
SELECT e.nombre, d.nombre
FROM empleados e
JOIN departamentos d ON d.id = e.departamento_id
WHERE e.salario > 2500;
```

En motores compatibles, `EXPLAIN ANALYZE` ejecuta la consulta y muestra tiempos reales. Úsalo con cuidado en `UPDATE`/`DELETE`. Revisa estimaciones de filas, scans completos, joins costosos, memoria y uso de índices.

---

# 18. Vistas y tablas temporales

## Vistas

Una vista guarda una consulta reutilizable:

```sql
CREATE VIEW empleados_activos AS
SELECT id, nombre, email, departamento_id
FROM empleados
WHERE activo = TRUE;

SELECT * FROM empleados_activos;

DROP VIEW empleados_activos;
```

Una vista normal no suele almacenar los datos. Una **vista materializada** sí guarda el resultado y necesita refrescarse; es útil para informes pesados cuando se tolera cierta falta de actualidad.

## Tablas temporales

```sql
CREATE TEMPORARY TABLE resumen_salarios AS
SELECT departamento_id, AVG(salario) AS media
FROM empleados
GROUP BY departamento_id;
```

Su duración y visibilidad dependen del motor y de la conexión.

---

# 19. Normalización y diseño relacional

La normalización reduce duplicidades y anomalías:

- **1FN:** valores atómicos, sin listas dentro de una celda.
- **2FN:** 1FN y cada atributo depende de toda la clave primaria, especialmente con claves compuestas.
- **3FN:** 2FN y los atributos no clave no dependen transitivamente de otros atributos no clave.
- **BCNF:** versión más estricta basada en dependencias funcionales.

Ejemplo: en lugar de guardar `telefonos = '600..., 611...'` en una columna, crea una tabla `telefonos_cliente(cliente_id, telefono)`.

A veces se **desnormaliza** por rendimiento, pero debe hacerse midiendo y documentando las reglas para mantener los datos consistentes.

### Relaciones

- 1 a 1: una fila se relaciona con una de otra tabla.
- 1 a N: un departamento tiene muchos empleados.
- N a N: alumnos y cursos; se resuelve con una tabla intermedia.

```sql
CREATE TABLE alumno_curso (
    alumno_id INTEGER REFERENCES alumnos(id),
    curso_id INTEGER REFERENCES cursos(id),
    matriculado_en DATE NOT NULL DEFAULT CURRENT_DATE,
    PRIMARY KEY (alumno_id, curso_id)
);
```

---

# 20. Permisos y seguridad

Aplica el principio de **mínimo privilegio**:

```sql
CREATE ROLE analista;
GRANT CONNECT ON DATABASE empresa TO analista;
GRANT USAGE ON SCHEMA public TO analista;
GRANT SELECT ON empleados, departamentos TO analista;
REVOKE INSERT, UPDATE, DELETE ON empleados FROM analista;
```

La sintaxis varía por SGBD. Recomendaciones:

- usa roles separados para lectura, escritura y administración;
- no guardes contraseñas en texto plano;
- cifra las conexiones cuando sea posible;
- registra accesos sensibles;
- limita datos mediante vistas o políticas de seguridad;
- no concedas permisos de administrador a la aplicación.

## Inyección SQL

Nunca construyas consultas concatenando datos del usuario:

```text
-- Peligroso: SELECT ... WHERE email = '" + entrada + "'
```

Usa consultas parametrizadas:

```text
SELECT id, nombre FROM usuarios WHERE email = ?
```

El placeholder puede ser `?`, `$1`, `:email` o `%s` según la biblioteca.

---

# 21. Procedimientos, funciones y triggers

Estos objetos son específicos del motor y suelen escribirse en un lenguaje propio (PL/pgSQL, T-SQL, PL/SQL…).

- **Función:** puede recibir parámetros y devolver un valor o conjunto de filas.
- **Procedimiento:** ejecuta acciones; puede permitir control transaccional según el motor.
- **Trigger:** se ejecuta automáticamente antes o después de un `INSERT`, `UPDATE` o `DELETE`.

Ejemplo conceptual de trigger:

```sql
-- La sintaxis concreta depende del SGBD
CREATE TRIGGER registrar_cambio
AFTER UPDATE ON empleados
FOR EACH ROW
EXECUTE FUNCTION guardar_auditoria();
```

No abuses de triggers: pueden ocultar cambios, dificultar las pruebas y complicar el rendimiento. Para auditoría, valida también quién puede modificar los datos y qué información debe conservarse.

---

# 22. JSON y datos semiestructurados

Cuando el motor lo soporta:

```sql
CREATE TABLE eventos (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    datos JSON NOT NULL
);

INSERT INTO eventos (datos)
VALUES ('{"tipo": "login", "usuario": "ana"}');
```

La extracción de campos cambia entre PostgreSQL, MySQL y SQL Server. Para datos muy consultados, considera columnas normales o índices específicos. JSON no sustituye automáticamente a un buen modelo relacional: usa cada formato para el tipo de datos adecuado.

---

# 23. Errores frecuentes

### 1. Olvidar `WHERE`

```sql
-- Actualizaría toda la tabla
UPDATE empleados SET salario = salario * 1.1;
```

Comprueba siempre el filtro con un `SELECT` y usa transacciones.

### 2. Confundir `WHERE` y `HAVING`

- `WHERE`: filtra filas antes de agrupar.
- `HAVING`: filtra grupos después de agrupar.

### 3. Usar `= NULL`

Usa `IS NULL` o `IS NOT NULL`.

### 4. Crear duplicados al hacer JOIN

Una relación 1:N multiplica las filas. Si solo quieres comprobar existencia, usa `EXISTS`; no añadas `DISTINCT` para ocultar un join incorrecto sin entender la causa.

### 5. Filtrar una ventana en WHERE

Usa una CTE o subconsulta.

### 6. Usar `NOT IN` con nulos

Prefiere `NOT EXISTS` cuando la subconsulta pueda devolver `NULL`.

### 7. Confiar en el orden sin `ORDER BY`

SQL no garantiza el orden de resultados si no lo solicitas.

### 8. Guardar fechas como texto

Usa `DATE`, `TIMESTAMP` o el tipo equivalente del motor.

### 9. Usar `FLOAT` para dinero

Usa `DECIMAL`/`NUMERIC`.

### 10. Confiar en `DISTINCT` para arreglar todo

`DISTINCT` elimina filas idénticas del resultado; no corrige relaciones mal planteadas.

---

# 24. Receta para construir una consulta

1. Define exactamente qué filas y columnas necesitas.
2. Identifica la tabla principal.
3. Añade `JOIN` con condiciones claras.
4. Filtra filas con `WHERE`.
5. Agrupa con `GROUP BY` si necesitas agregados.
6. Filtra grupos con `HAVING`.
7. Añade ventanas si necesitas cálculos sin perder detalle.
8. Ordena con `ORDER BY`.
9. Limita resultados si procede.
10. Prueba casos con `NULL`, duplicados, empates y tablas vacías.
11. Revisa el plan con `EXPLAIN` si el volumen es importante.
12. Parametriza toda entrada externa.

## Plantilla general

```sql
SELECT
    columnas,
    agregados,
    funciones_de_ventana
FROM tabla_principal AS t
[INNER | LEFT | RIGHT | FULL] JOIN otra_tabla AS o
    ON condicion_de_relacion
WHERE filtros_de_filas
GROUP BY columnas_no_agregadas
HAVING filtros_de_grupos
ORDER BY columnas
LIMIT cantidad OFFSET desplazamiento;
```

---

# 25. Resumen rápido: ¿qué herramienta uso?

| Necesidad | Herramienta |
|---|---|
| Seleccionar filas | `SELECT ... WHERE` |
| Evitar duplicados del resultado | `DISTINCT` |
| Unir tablas | `JOIN` |
| Calcular por grupos y reducir filas | `GROUP BY` |
| Filtrar grupos | `HAVING` |
| Calcular por grupo manteniendo cada fila | función de ventana + `OVER` |
| Filtrar el resultado de una ventana | CTE o subconsulta |
| Reutilizar una consulta legible | `WITH` |
| Recorrer jerarquías | `WITH RECURSIVE` |
| Comprobar existencia | `EXISTS` |
| Combinar listas compatibles | `UNION`, `INTERSECT`, `EXCEPT` |
| Insertar | `INSERT` |
| Modificar | `UPDATE` |
| Borrar | `DELETE` |
| Cambiar la estructura | `ALTER TABLE` |
| Guardar varios cambios como unidad | transacción (`BEGIN`, `COMMIT`, `ROLLBACK`) |
| Acelerar consultas | índices + `EXPLAIN` |
| Reutilizar una consulta | vista |
| Controlar acceso | roles, `GRANT`, `REVOKE` |

> **Idea clave:** una subconsulta resuelve un cálculo dentro de otra consulta; una CTE organiza consultas complejas; un `JOIN` combina tablas; `GROUP BY` reduce filas para obtener resúmenes; una función de ventana calcula sobre filas relacionadas sin eliminarlas; y las transacciones garantizan que los cambios importantes sean seguros.

---

## Nota final sobre dialectos

SQL tiene un estándar, pero cada motor añade funciones y variaciones. Antes de copiar una consulta, comprueba la documentación de tu SGBD para `LIMIT/TOP/FETCH`, identidad/autoincremento, fechas, `UPSERT`, JSON, funciones de texto, procedimientos, triggers y tipos de aislamiento. Aprende primero la lógica relacional y después la sintaxis concreta del motor que utilices.
