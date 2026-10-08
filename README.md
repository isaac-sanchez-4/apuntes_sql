# Apuntes de SQL — Guía de sintaxis y estructura

Guía de referencia rápida en español, consolidada a partir de los materiales del repositorio. Está enfocada en **sintaxis, componentes, reglas y variantes por motor**, no en colecciones de ejercicios resueltos.

## Tabla de contenidos

- [1. Alcance y enfoque](#sec-1)
- [2. Qué es SQL y tipos de sentencias](#sec-2)
- [3. SELECT: estructura base de consulta](#sec-3)
- [4. WHERE y operadores de filtrado](#sec-4)
- [5. ORDER BY, DISTINCT, LIMIT/OFFSET/TOP](#sec-5)
- [6. Agregaciones, GROUP BY y HAVING](#sec-6)
- [7. JOINs y combinación de tablas](#sec-7)
- [8. Subconsultas, EXISTS y CTE (`WITH`)](#sec-8)
- [9. Operadores de conjunto](#sec-9)
- [10. INSERT, UPDATE y DELETE](#sec-10)
- [11. DDL: crear y modificar bases/tablas](#sec-11)
- [12. Tipos de datos, restricciones y claves](#sec-12)
- [13. Índices](#sec-13)
- [14. Vistas](#sec-14)
- [15. Funciones de ventana](#sec-15)
- [16. Funciones, procedimientos y triggers](#sec-16)
- [17. Transacciones](#sec-17)
- [18. Normalización y diseño relacional](#sec-18)
- [19. Notas de compatibilidad entre motores](#sec-19)
- [20. Errores frecuentes vistos en el material](#sec-20)
- [21. Referencias y créditos](#sec-21)

---

<a id="sec-1"></a>
## 1. Alcance y enfoque

- Cobertura principal: `SELECT`, filtros, ordenación, agregación, joins, subconsultas, operadores de conjunto, DML, DDL, claves, restricciones, índices, vistas, transacciones, funciones de ventana y conceptos de diseño.
- Se prioriza **sintaxis mínima reusable** con marcadores como `<tabla>`, `<columna>`, `<condición>`.
- Se distinguen reglas generales (SQL estándar) y variaciones de motor cuando aparecen en los materiales.

<a id="sec-2"></a>
## 2. Qué es SQL y tipos de sentencias

SQL (Structured Query Language) es el lenguaje de trabajo de bases de datos relacionales.

Familias de sentencias:

- **DQL** (consulta): `SELECT`
- **DML** (datos): `INSERT`, `UPDATE`, `DELETE`
- **DDL** (estructura): `CREATE`, `ALTER`, `DROP`, `TRUNCATE`
- **TCL** (transacciones): `BEGIN/START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`
- **DCL** (permisos, complementario): `GRANT`, `REVOKE`

<a id="sec-3"></a>
## 3. SELECT: estructura base de consulta

Propósito: consultar datos de una o más tablas.

Sintaxis general:

```sql
SELECT [DISTINCT] <selección>
FROM <tabla_o_fuente> [AS <alias>]
[JOIN <tabla> ON <condición_join> ...]
[WHERE <condición_filas>]
[GROUP BY <columnas>]
[HAVING <condición_grupos>]
[ORDER BY <columna|expresión> [ASC|DESC] ...]
[LIMIT <n> [OFFSET <m>] | FETCH ... | TOP <n>];
```

Partes relevantes:

- `<selección>` puede contener columnas, alias, expresiones, `CASE`, agregaciones o ventanas.
- `DISTINCT` elimina duplicados del resultado final.
- `*` es útil para exploración; para trabajo real conviene listar columnas.

Orden lógico de evaluación (resumido):

1. `FROM` / `JOIN`
2. `WHERE`
3. `GROUP BY`
4. `HAVING`
5. `SELECT`
6. `ORDER BY`
7. `LIMIT/OFFSET` (o equivalente)

<a id="sec-4"></a>
## 4. WHERE y operadores de filtrado

Propósito: filtrar filas antes de agrupar.

Sintaxis:

```sql
WHERE <condición_booleana>
```

Operadores y predicados frecuentes en el material:

- Comparación: `=`, `<>`, `>`, `>=`, `<`, `<=`
- Lógicos: `AND`, `OR`, `NOT`
- Rango: `BETWEEN <a> AND <b>`
- Conjunto: `IN (<lista|subconsulta>)`
- Patrón: `LIKE`, `NOT LIKE` (`%` y `_`)
- Nulos: `IS NULL`, `IS NOT NULL`
- Existencia: `EXISTS`, `NOT EXISTS`
- Cuantificados: `ANY`, `ALL`

Reglas:

- `NULL` no se compara con `=`; usar `IS NULL` / `IS NOT NULL`.
- Si se combinan `AND` y `OR`, usar paréntesis para evitar ambigüedad.

<a id="sec-5"></a>
## 5. ORDER BY, DISTINCT, LIMIT/OFFSET/TOP

### ORDER BY

Propósito: ordenar el resultado.

```sql
ORDER BY <columna|expresión> [ASC|DESC], ...
```

- `ASC` (por defecto), `DESC`.
- El orden no está garantizado si no se declara `ORDER BY`.

### DISTINCT

Propósito: deduplicar filas del resultado proyectado.

```sql
SELECT DISTINCT <columnas>
FROM <tabla>;
```

### LIMIT/OFFSET/TOP

Propósito: recortar resultados o paginar.

```sql
-- PostgreSQL/MySQL/SQLite
LIMIT <n> [OFFSET <m>]

-- SQL Server
SELECT TOP (<n>) <columnas>
FROM <tabla>;
```

Nota: en paginación, combinar con `ORDER BY` para resultados deterministas.

<a id="sec-6"></a>
## 6. Agregaciones, GROUP BY y HAVING

Propósito: resumir filas en grupos.

Funciones presentes en los materiales:

- `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`

Sintaxis:

```sql
SELECT <columna_grupo>, <agregación(...)>
FROM <tabla>
[WHERE <filtro_filas>]
GROUP BY <columna_grupo>
[HAVING <filtro_sobre_agregación>];
```

Reglas:

- `WHERE` filtra filas antes del agrupamiento.
- `HAVING` filtra grupos después del agrupamiento.
- Toda columna no agregada del `SELECT` debe estar en `GROUP BY` (regla general).

<a id="sec-7"></a>
## 7. JOINs y combinación de tablas

Propósito: relacionar tablas por claves.

Sintaxis base:

```sql
FROM <tabla_a> AS a
<tipo_join> <tabla_b> AS b
  ON <condición_relación>
```

Tipos mencionados:

- `INNER JOIN`: solo coincidencias.
- `LEFT JOIN`: conserva todas las filas de la izquierda.
- `RIGHT JOIN`: conserva todas las de la derecha (si el motor lo soporta).
- `FULL [OUTER] JOIN`: conserva filas de ambos lados (si lo soporta).
- `CROSS JOIN`: producto cartesiano.
- `SELF JOIN`: unión de una tabla consigo misma.

Reglas prácticas:

- Expresar siempre el criterio de relación en `ON`.
- En `LEFT JOIN`, mover al `ON` los filtros de la tabla derecha cuando quieras preservar no-coincidentes.

<a id="sec-8"></a>
## 8. Subconsultas, EXISTS y CTE (`WITH`)

### Subconsultas

Pueden aparecer en:

- `WHERE` (comparación, `IN`, `EXISTS`)
- `SELECT` (subconsulta escalar)
- `FROM` (tabla derivada)

Patrones:

```sql
WHERE <columna> IN (SELECT <columna> FROM <tabla> ...)

WHERE EXISTS (
  SELECT 1
  FROM <tabla_relacionada>
  WHERE <condición_correlacionada>
)
```

`NOT EXISTS` se usa en el material como anti-join para detectar ausencias.

### CTE (`WITH`)

Propósito: nombrar resultados intermedios y mejorar legibilidad.

```sql
WITH <cte> AS (
  SELECT ...
)
SELECT ...
FROM <cte>;
```

Recursiva (estructura jerárquica):

```sql
WITH RECURSIVE <cte> AS (
  <consulta_ancla>
  UNION ALL
  <consulta_recursiva>
)
SELECT ...
FROM <cte>;
```

<a id="sec-9"></a>
## 9. Operadores de conjunto

Propósito: combinar resultados compatibles (mismo número de columnas y tipos compatibles).

```sql
<consulta_1>
UNION [ALL]
<consulta_2>

<consulta_1>
INTERSECT
<consulta_2>

<consulta_1>
EXCEPT
<consulta_2>
```

- `UNION`: elimina duplicados.
- `UNION ALL`: conserva duplicados.
- `INTERSECT`: intersección.
- `EXCEPT`: diferencia (primer conjunto menos segundo).

<a id="sec-10"></a>
## 10. INSERT, UPDATE y DELETE

### INSERT

```sql
INSERT INTO <tabla> (<col1>, <col2>, ...)
VALUES (<val1>, <val2>, ...);

INSERT INTO <tabla_destino> (<col...>)
SELECT <col...>
FROM <tabla_origen>
WHERE <condición>;
```

### UPDATE

```sql
UPDATE <tabla>
SET <col1> = <expr1>, <col2> = <expr2>, ...
[WHERE <condición>];
```

### DELETE

```sql
DELETE FROM <tabla>
[WHERE <condición>];
```

Regla crítica: validar primero el `WHERE` (por ejemplo con `SELECT`) para evitar cambios masivos no deseados.

<a id="sec-11"></a>
## 11. DDL: crear y modificar bases/tablas

### Base de datos

```sql
CREATE DATABASE <base_datos>;
```

### Tablas

```sql
CREATE TABLE <tabla> (
  <columna> <tipo> [restricciones_columna],
  ...,
  [restricciones_tabla]
);
```

### Cambios estructurales

```sql
ALTER TABLE <tabla>
  ADD <definición_columna|restricción>;

ALTER TABLE <tabla>
  ALTER COLUMN <columna> <acción>;

ALTER TABLE <tabla>
  DROP COLUMN <columna>;

DROP TABLE [IF EXISTS] <tabla>;
```

Nota: la sintaxis exacta de `ALTER COLUMN`, `IF EXISTS`, renombrados o tipos puede variar entre motores.

<a id="sec-12"></a>
## 12. Tipos de datos, restricciones y claves

### Tipos de datos comunes

- Numéricos enteros: `SMALLINT`, `INT/INTEGER`, `BIGINT`
- Numéricos decimales: `DECIMAL(p,s)`, `NUMERIC(p,s)`
- Texto: `CHAR(n)`, `VARCHAR(n)`, `TEXT`
- Fecha/hora: `DATE`, `TIME`, `TIMESTAMP`
- Booleano: `BOOLEAN` (o equivalente)

### Restricciones

- `NOT NULL`
- `UNIQUE`
- `CHECK (<condición>)`
- `DEFAULT <valor|expresión>`
- `PRIMARY KEY (...)`
- `FOREIGN KEY (...) REFERENCES <tabla>(...) [ON UPDATE ...] [ON DELETE ...]`

Claves:

- **Primaria**: identifica cada fila (simple o compuesta).
- **Foránea**: mantiene integridad referencial entre tablas.

<a id="sec-13"></a>
## 13. Índices

Propósito: acelerar búsqueda, join y ordenación (a costa de espacio y coste de escritura).

Sintaxis base:

```sql
CREATE [UNIQUE] INDEX <idx_nombre>
ON <tabla> (<columna1> [ASC|DESC], ...);
```

Regla general: indexar columnas usadas frecuentemente en `WHERE`, `JOIN`, `ORDER BY` y claves foráneas.

<a id="sec-14"></a>
## 14. Vistas

Propósito: encapsular una consulta reutilizable.

```sql
CREATE [OR ALTER] VIEW <vista> AS
SELECT ...;

-- lectura
SELECT ... FROM <vista>;

-- eliminación
DROP VIEW <vista>;
```

Nota del material: el orden final debe aplicarse en la consulta de consumo (`SELECT ... FROM <vista> ORDER BY ...`), no confiar en `ORDER BY` interno de la vista.

<a id="sec-15"></a>
## 15. Funciones de ventana

Propósito: calcular métricas por particiones sin colapsar filas.

Sintaxis general:

```sql
<función_ventana>() OVER (
  [PARTITION BY <columna...>]
  [ORDER BY <columna...>]
)
```

Funciones tratadas:

- `ROW_NUMBER()`
- `RANK()`
- `DENSE_RANK()`
- Uso de agregaciones con `OVER (...)` como `AVG(...) OVER (...)`

<a id="sec-16"></a>
## 16. Funciones, procedimientos y triggers

Estos objetos aparecen como tema en el material de apoyo y son dependientes del motor.

- **Función**: devuelve valor o conjunto.
- **Procedimiento**: ejecuta lógica procedural.
- **Trigger**: dispara lógica ante `INSERT`/`UPDATE`/`DELETE`.

Plantilla conceptual de trigger:

```sql
CREATE TRIGGER <nombre>
<BEFORE|AFTER> <INSERT|UPDATE|DELETE>
ON <tabla>
FOR EACH ROW
<acción_dependiente_del_motor>;
```

<a id="sec-17"></a>
## 17. Transacciones

Propósito: agrupar operaciones con confirmación o reversión.

```sql
BEGIN; -- o START TRANSACTION
  <sentencia_1>;
  <sentencia_2>;
COMMIT;

-- en caso de error
ROLLBACK;
```

Con puntos intermedios:

```sql
SAVEPOINT <nombre>;
ROLLBACK TO SAVEPOINT <nombre>;
RELEASE SAVEPOINT <nombre>;
```

<a id="sec-18"></a>
## 18. Normalización y diseño relacional

Conceptos presentes:

- **1FN**: valores atómicos.
- **2FN**: dependencia completa de la clave.
- **3FN**: sin dependencias transitivas no clave.
- Relaciones: 1:1, 1:N, N:M (esta última mediante tabla intermedia con claves foráneas).

Objetivo: reducir redundancia y anomalías de inserción/actualización/borrado.

<a id="sec-19"></a>
## 19. Notas de compatibilidad entre motores

Puntos recurrentes:

- Límite de filas: `LIMIT/OFFSET` (PostgreSQL/MySQL/SQLite) vs `TOP` (SQL Server).
- Vistas: `CREATE OR ALTER VIEW` es típico en SQL Server; en otros motores puede cambiar.
- Tipos y autoincrementos (`IDENTITY`, `AUTO_INCREMENT`, `SERIAL`, etc.) varían por motor.
- Soporte de `RIGHT/FULL JOIN`, recursión, `INTERSECT/EXCEPT` y detalles de `ALTER TABLE` depende del SGBD.

<a id="sec-20"></a>
## 20. Errores frecuentes vistos en el material

- Confundir `WHERE` (filas) con `HAVING` (grupos).
- Comparar con `NULL` usando `=` en vez de `IS NULL`.
- Usar `ORDER BY` en lugares no aplicables o confiar en orden implícito.
- Olvidar `WHERE` en `UPDATE`/`DELETE`.
- Convertir accidentalmente un `LEFT JOIN` en `INNER` por filtrar mal.
- Usar `NOT IN` sin considerar `NULL` en subconsultas.

<a id="sec-21"></a>
## 21. Referencias y créditos

Material base revisado y consolidado:

- `EJERCICIOS CALSE14 (1).docx`
- `EJERCICIOS CALSE14 (1).pdf`
- `Ejercicio 1 (1).docx`
- `Ejercicio 1 (1).pdf`
- `datacamp_sql_master_cheatsheet (1).pdf`
- `practica8_ISAAC_Sanchez (1).docx`
- `practica8_ISAAC_Sanchez (1).pdf`

Material complementario aprovechado:

- `sql.md`
- `respuestas (1) (1).md`

---
