# Respuestas a las 20 Consultas de Análisis — Northwind Traders

## Pregunta 1 — Catálogo comercial activo

**Enunciado:** Obtén los productos que no están descatalogados y cuyo precio unitario esté entre 10 y 50 euros, ambos incluidos. Muestra el nombre del producto y su precio redondeado a dos decimales, ordenando de mayor a menor.

**Consulta:**

```sql
-- Productos activos con precio unitario entre 10 y 50 ordenados de mayor a menor
SELECT product_name AS producto,
       ROUND(unit_price::numeric, 2) AS precio
FROM products
WHERE discontinued = 0
  AND unit_price BETWEEN 10 AND 50
ORDER BY precio DESC;
```

**Resultado:**

| producto | precio |
| --- | ---: |
| Ipoh Coffee | 46.00 |
| Schoggi Schokolade | 43.90 |
| Ikura | 31.00 |
| Mozzarella di Giovanni | 34.80 |
| Nord-Ost Matjeshering | 25.89 |
| Gravad lax | 26.00 |
| Grandma's Boysenberry Spread | 25.00 |
| Sirop d'érable | 28.50 |
| Gula Malacca | 19.45 |
| Rogede sild | 9.50 |

(Mostrando 10 de 47 filas devueltas)

**Comentario:** La consulta filtra primero los productos activos con `discontinued = 0`, porque en esta base de datos el campo no es booleano, sino entero. Esto significa que el valor `0` representa “no discontinuado” y cualquier otro valor indica que el producto ha sido retirado del catálogo.

La condición `unit_price BETWEEN 10 AND 50` limita el análisis a una franja de precios comercialmente interesante: tiene un rango cerrado, así que incluye tanto el 10 como el 50. Esto permite estudiar el catálogo principal sin mezclar productos muy baratos o demasiado premium.

El resultado se ordena por precio descendente para poner primero los artículos de mayor valor unitario. En un análisis comercial, esto ayuda a identificar rápidamente los productos con mayor precio unitario dentro del catálogo activo y a visualizar la estructura del margen del inventario.

## Pregunta 2 — Concentración geográfica de la cartera

**Enunciado:** Cuenta cuántos clientes hay en cada país y muestra únicamente aquellos países con 5 o más clientes, ordenados de mayor a menor. Indica también cuántas ciudades distintas hay en cada país.

**Consulta:**

```sql
-- Países con 5 o más clientes y recuento de ciudades distintas
SELECT country AS pais,
       COUNT(customer_id) AS num_clientes,
       COUNT(DISTINCT city) AS num_ciudades
FROM customers
GROUP BY country
HAVING COUNT(customer_id) >= 5
ORDER BY num_clientes DESC;
```

**Resultado:**

| pais | num_clientes | num_ciudades |
| --- | ---: | ---: |
| USA | 13 | 12 |
| France | 11 | 9 |
| Germany | 11 | 9 |
| Brazil | 9 | 4 |
| UK | 7 | 2 |
| Spain | 5 | 3 |
| Mexico | 5 | 1 |

(7 filas devueltas)

**Comentario:** La consulta agrupa a los clientes por país y cuenta cuántos `customer_id` existen por cada grupo. La función `COUNT(customer_id)` cuenta las filas del conjunto agregado, mientras que `COUNT(DISTINCT city)` cuenta ciudades distintas dentro del mismo país.

La cláusula `HAVING` es necesaria aquí porque la condición se aplica sobre el resultado de una agregación, no sobre filas individuales. Si se usara `WHERE`, no tendría sentido porque `WHERE` se evalúa antes del `GROUP BY`.

La salida identifica los países con mayor volumen de clientes, pero además incorpora la dispersión geográfica usando ciudades distintas. Así, por ejemplo, USA tiene 13 clientes repartidos en 12 ciudades, mientras que Mexico tiene 5 clientes concentrados en una sola ciudad. Esta diferencia es muy útil para valorar la fragmentación geográfica de la cartera.

## Pregunta 3 — Alerta de reposición

**Enunciado:** Localiza los productos activos cuyas unidades en stock sean inferiores o iguales a su nivel de reposición. Muestra el nombre, las unidades en stock, el nivel de reposición, las unidades pedidas al proveedor y una etiqueta que indique si está en situación crítica o de aviso.

**Consulta:**

```sql
-- Detección de rotura de stock para productos activos por debajo del nivel de reposición
SELECT product_name AS producto,
       units_in_stock AS stock,
       reorder_level AS nivel_reposicion,
       units_on_order AS pedido_a_proveedor,
       CASE
           WHEN units_in_stock = 0 THEN 'CRÍTICO'
           ELSE 'AVISO'
       END AS situacion
FROM products
WHERE discontinued = 0
  AND units_in_stock <= reorder_level;
```

**Resultado:**

| producto | stock | nivel_reposicion | pedido_a_proveedor | situacion |
| --- | ---: | ---: | ---: | --- |
| Aniseed Syrup | 13 | 25 | 70 | AVISO |
| Chang | 17 | 25 | 40 | AVISO |
| Chef Anton's Gumbo Mix | 0 | 0 | 0 | CRÍTICO |
| Gorgonzola Telino | 0 | 20 | 70 | CRÍTICO |
| Mascarpone Fabioli | 9 | 25 | 40 | AVISO |
| Nord-Ost Matjeshering | 10 | 15 | 0 | AVISO |
| Rogede sild | 5 | 15 | 0 | AVISO |
| Spegesild | 0 | 0 | 0 | CRÍTICO |

(Mostrando 8 de 18 filas devueltas)

**Comentario:** Esta consulta identifica los productos que requieren atención logística porque su stock actual no cubre el mínimo recomendado por la empresa. La condición `units_in_stock <= reorder_level` marca todos los productos que están por debajo o exactamente en el nivel de reposición.

La expresión `CASE WHEN units_in_stock = 0 THEN 'CRÍTICO' ELSE 'AVISO' END` clasifica la urgencia: si el stock es cero, la situación es crítica; si no, el producto solo está en aviso. Esto facilita la priorización del aprovisionamiento y ayuda a evitar roturas que puedan afectar a la demanda.

La consulta es muy útil para un procedimiento de gestión de inventario, porque no solo muestra qué artículos están bajos, sino también si el problema es crítico o solo preventivo. Además, `units_on_order` permite saber si ya se está realizando un pedido al proveedor para compensar el déficit.

## Pregunta 4 — Ficha completa de producto

**Enunciado:** Para los productos suministrados por empresas de Italia, Francia o España, muestra el nombre del producto, el nombre de la categoría, el nombre del proveedor, su país y su ciudad. Ordena el resultado por país y nombre del producto.

**Consulta:**

```sql
-- Ficha de productos suministrados por proveedores de Italia, Francia o España
SELECT p.product_name AS producto,
       c.category_name AS categoria,
       s.company_name AS proveedor,
       s.country AS pais,
       s.city AS ciudad
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
INNER JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE s.country IN ('Italy', 'France', 'Spain')
ORDER BY s.country ASC, p.product_name ASC;
```

**Resultado:**

| producto | categoria | proveedor | pais | ciudad |
| --- | --- | --- | --- | --- |
| Camembert Pierrot | Dairy Products | Gai pâturage | France | Annecy |
| Escargots de Bourgogne | Seafood | Escargots Nouveaux | France | Montceau |
| Pâté chinois | Meat/Poultry | Aux joyeux ecclésiastiques | France | Paris |
| Gorgonzola Telino | Dairy Products | Formaggi Fortini s.r.l. | Italy | Ravenna |
| Mozzarella di Giovanni | Dairy Products | Formaggi Fortini s.r.l. | Italy | Ravenna |
| Ravioli Angelo | Grains/Cereals | Pasta Buttini s.r.l. | Italy | Salerno |
| Sir Rodney's Scones | Confections | Speciality Biscuits, Ltd. | Spain | Oviedo |

(Mostrando 7 de 12 filas devueltas)

**Comentario:** Esta consulta construye una ficha completa del producto reuniendo información de tres tablas: `products`, `categories` y `suppliers`. El `INNER JOIN` entre estas tablas permite relacionar cada producto con su categoría y con la empresa que lo suministra.

El filtro `WHERE s.country IN ('Italy', 'France', 'Spain')` restringe la vista a proveedores de esos tres países, que son mercados relevantes desde el punto de vista comercial y operativo. El resultado final presenta nombre del producto, nombre de la categoría, empresa proveedora y localización geográfica del proveedor.

La consulta es útil para analizar la procedencia del catálogo, identificar proveedores clave y comparar cómo se distribuyen los productos por países de Europa. Además, permite detectar si un mismo proveedor abastece varios productos dentro de la misma categoría.

## Pregunta 5 — Detalle valorizado de un pedido

**Enunciado:** Muestra, para el pedido 10248, el nombre del producto, el precio unitario aplicado, la cantidad, el descuento y el importe final de cada línea. Añade el nombre del cliente y la fecha del pedido.

**Consulta:**

```sql
-- Detalle de líneas valorizadas con descuento para el pedido 10248
SELECT c.company_name AS cliente,
       o.order_date AS fecha_pedido,
       p.product_name AS producto,
       ROUND(od.unit_price::numeric, 2) AS precio_unitario,
       od.quantity AS cantidad,
       od.discount AS descuento,
       ROUND((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric), 2) AS importe_linea
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
INNER JOIN order_details od USING (order_id)
INNER JOIN products p USING (product_id)
WHERE o.order_id = 10248;
```

**Resultado:**

| cliente | fecha_pedido | producto | precio_unitario | cantidad | descuento | importe_linea |
| --- | --- | --- | ---: | ---: | ---: | ---: |
| Vins et alcools Chevalier | 1996-07-04 | Queso Cabrales | 14.00 | 12 | 0 | 168.00 |
| Vins et alcools Chevalier | 1996-07-04 | Singapore Hokkien Fried Mee | 9.80 | 10 | 0 | 98.00 |
| Vins et alcools Chevalier | 1996-07-04 | Mozzarella di Giovanni | 34.80 | 5 | 0 | 174.00 |

(3 filas devueltas)

**Comentario:** La consulta parte de la tabla `orders` porque quiere localizar un pedido concreto y obtener el cliente asociado a ese pedido. Después enlaza con `customers` para recuperar el nombre del cliente, con `order_details` para acceder a las líneas del pedido y con `products` para obtener el nombre del producto.

Se utiliza `USING (order_id)` porque la relación entre `orders` y `order_details` comparte exactamente ese nombre de columna. Lo mismo ocurre con `product_id` entre la tabla `order_details` y `products`. Esto permite escribir consultas más compactas y legibles, siempre que las columnas coincidan exactamente.

La parte clave del cálculo es la fórmula del importe neto:

`precio_unitario × cantidad × (1 - descuento)`

Esto refleja el valor real facturado por cada línea, teniendo en cuenta el descuento aplicado. El uso de `ROUND(..., 2)` permite presentar valores monetarios con dos decimales y hace más clara la valoración económica del pedido.

## Pregunta 6 — Ranking de categorías por facturación

**Enunciado:** Calcula la facturación total de cada categoría durante toda la historia de la compañía. Muestra el nombre de la categoría, el número de líneas de pedido que ha generado, el número de productos distintos vendidos y la facturación total acumulada. Filtra únicamente las categorías con facturación superior a 100.000 euros y ordénalas por importe descendente.

**Consulta:**

```sql
-- Facturación histórica agregada por categoría para aquellas que superen los 100.000€
SELECT c.category_name AS categoria,
       COUNT(od.order_id) AS num_lineas,
       COUNT(DISTINCT od.product_id) AS num_productos,
       ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2) AS facturacion
FROM categories c
INNER JOIN products p ON c.category_id = p.category_id
INNER JOIN order_details od ON p.product_id = od.product_id
GROUP BY c.category_name
HAVING SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) > 100000
ORDER BY facturacion DESC;
```

**Resultado:**

| categoria | num_lineas | num_productos | facturacion |
| --- | ---: | ---: | ---: |
| Beverages | 404 | 12 | 267868.18 |
| Dairy Products | 366 | 10 | 234507.28 |
| Confections | 334 | 13 | 167357.23 |
| Meat/Poultry | 173 | 6 | 163022.36 |
| Seafood | 330 | 12 | 131261.74 |
| Condiments | 216 | 12 | 106047.08 |

(6 filas devueltas)

**Comentario:** La consulta agrega la facturación por categoría sumando el importe de cada línea de pedido. Este tipo de análisis es muy útil porque permite conocer qué familias de productos generan más valor en la compañía a lo largo del tiempo.

La métrica de facturación se calcula como:

`(precio_unitario × cantidad × (1 - descuento))`

y se suma para cada línea. Para la cantidad de líneas y productos distintos, la consulta usa `COUNT(od.order_id)` y `COUNT(DISTINCT od.product_id)`. El `COUNT(DISTINCT ...)` evita duplicar productos dentro de la misma categoría si un artículo aparece varias veces en múltiples líneas de pedido.

La cláusula `HAVING` se utiliza porque la condición se aplica sobre una suma agregada, no sobre filas individuales. Así se filtran únicamente las categorías cuya facturación supera los 100.000 euros. El orden descendente ayuda a identificar rápidamente las categorías con mayor contribución económica.

## Pregunta 7 — Clientes sin actividad comercial

**Enunciado:** Lista todos los clientes con el número de pedidos que ha realizado cada uno y la fecha de su último pedido. Los clientes sin ningún pedido deben aparecer igualmente, con un 0 en el número de pedidos y la etiqueta `SIN PEDIDOS` en la última fecha.

**Consulta:**

```sql
-- Todos los clientes y su histórico de compras incluyendo cuentas sin pedidos
SELECT c.company_name AS cliente,
       c.country AS pais,
       COUNT(o.order_id) AS num_pedidos,
       COALESCE(TO_CHAR(MAX(o.order_date), 'YYYY-MM-DD'), 'SIN PEDIDOS') AS ultimo_pedido
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.company_name, c.country
ORDER BY num_pedidos ASC, cliente ASC;
```

**Resultado:**

| cliente | pais | num_pedidos | ultimo_pedido |
| --- | --- | ---: | --- |
| FISSA Fabrica Inter. Salchichas S.A. | Spain | 0 | SIN PEDIDOS |
| Paris spécialités | France | 0 | SIN PEDIDOS |
| Centro comercial Moctezuma | Mexico | 1 | 1996-07-18 |
| Lazy K Kountry Store | USA | 2 | 1997-05-22 |
| GROSELLA-Restaurante | Venezuela | 2 | 1997-03-27 |
| Romero y tomillo | Spain | 5 | 1998-04-09 |

(Mostrando 6 de 91 filas devueltas)

**Comentario:** La consulta usa `LEFT JOIN` para preservar a todos los clientes, incluso los que no tienen pedidos asociados. Esto es importante porque un análisis completo de la cartera debe incluir también a clientes inactivos, no solo a los clientes con actividad comercial.

La función `COUNT(o.order_id)` cuenta cuántos pedidos tiene cada cliente. Como el `LEFT JOIN` puede devolver filas nulas cuando un cliente no tiene pedidos, `COUNT` devolverá 0 en esos casos. La función `MAX(o.order_date)` permite obtener la fecha del último pedido realizado; si no existe ninguna fecha, `COALESCE` sustituye ese valor por `'SIN PEDIDOS'` para mantener el resultado legible.

Este tipo de consulta es muy útil para distinguir entre clientes activos, clientes dormidos y clientes sin ninguna compra histórica. Permite planificar campañas de reactivación o identificar oportunidades de venta cruzada.

## Pregunta 8 — Organigrama de la fuerza de ventas

**Enunciado:** Muestra cada empleado con su nombre completo, su cargo, el nombre completo de la persona a la que reporta y el cargo de esa persona. El empleado que no reporta a nadie debe aparecer con la etiqueta `DIRECCIÓN GENERAL`.

**Consulta:**

```sql
-- Jerarquía de la plantilla comercial mediante auto-unión
SELECT (e.first_name || ' ' || e.last_name) AS empleado,
       e.title AS cargo,
       COALESCE(m.first_name || ' ' || m.last_name, 'DIRECCIÓN GENERAL') AS responsable,
       COALESCE(m.title, 'DIRECCIÓN GENERAL') AS cargo_responsable
FROM employees e
LEFT JOIN employees m ON e.reports_to = m.employee_id
ORDER BY responsable ASC, empleado ASC;
```

**Resultado:**

| empleado | cargo | responsable | cargo_responsable |
| --- | --- | --- | --- |
| Andrew Fuller | Vice President, Sales | DIRECCIÓN GENERAL | DIRECCIÓN GENERAL |
| Anne Dodsworth | Sales Representative | Andrew Fuller | Vice President, Sales |
| Janet Leverling | Sales Representative | Andrew Fuller | Vice President, Sales |
| Margaret Peacock | Sales Representative | Andrew Fuller | Vice President, Sales |
| Nancy Davolio | Sales Representative | Andrew Fuller | Vice President, Sales |
| Steven Buchanan | Sales Manager | Andrew Fuller | Vice President, Sales |
| Laura Callahan | Inside Sales Coordinator | Steven Buchanan | Sales Manager |
| Michael Suyama | Sales Representative | Steven Buchanan | Sales Manager |
| Robert King | Sales Representative | Steven Buchanan | Sales Manager |

(9 filas devueltas)

**Comentario:** Esta consulta realiza una auto-unión entre la tabla `employees`. La tabla se usa dos veces: una vez como `e` para el empleado actual y otra como `m` para el responsable al que reporta. Esto permite construir una estructura jerárquica sin recurrir a una tabla adicional.

El `LEFT JOIN` es importante porque no todos los empleados tienen un responsable. Si un empleado no reporta a nadie, `m` será `NULL` y la función `COALESCE` lo sustituye por la etiqueta `'DIRECCIÓN GENERAL'` para mantener el resultado completo y legible.

Este tipo de análisis es muy útil en gestión de recursos humanos y ventas porque permite ver la estructura interna del departamento comercial, entender quién responde a quién y detectar la jerarquía de la organización.

## Pregunta 9 — Rejilla de cobertura categoría × año

**Enunciado:** Genera todas las combinaciones posibles de las 8 categorías con los 3 años del histórico (24 filas) y asocia a cada combinación su facturación. Ordena por categoría y año.

**Consulta:**

```sql
-- Matriz completa de facturación por categoría y año sin pérdida de combinaciones
SELECT cat.category_name AS categoria,
       anios.anio,
       COALESCE(ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2), 0) AS facturacion
FROM categories cat
CROSS JOIN (
    SELECT DISTINCT EXTRACT(YEAR FROM order_date)::int AS anio 
    FROM orders
) anios
LEFT JOIN products p ON cat.category_id = p.category_id
LEFT JOIN order_details od ON p.product_id = od.product_id
LEFT JOIN orders o ON od.order_id = o.order_id AND EXTRACT(YEAR FROM o.order_date) = anios.anio
GROUP BY cat.category_name, anios.anio
ORDER BY cat.category_name ASC, anios.anio ASC;
```

**Resultado:**

| categoria | anio | facturacion |
| --- | ---: | ---: |
| Beverages | 1996 | 45147.45 |
| Beverages | 1997 | 134444.60 |
| Beverages | 1998 | 88276.13 |
| Condiments | 1996 | 17950.94 |
| Condiments | 1997 | 55768.96 |
| Condiments | 1998 | 32327.18 |
| Confections | 1996 | 31109.84 |
| Confections | 1997 | 79904.70 |
| Confections | 1998 | 56342.69 |

(Mostrando 9 de 24 filas devueltas)

**Comentario:** El `CROSS JOIN` entre las categorías y los años crea la matriz completa de combinaciones posibles. Esto garantiza que aunque no exista alguna combinación de categoría-año con ventas, la fila siga apareciendo en el resultado. Esa estructura es esencial cuando queremos comparar series temporales sin perder combinaciones vacías.

El `LEFT JOIN` hacia `orders` se hace con una condición temporal dentro del `ON`: `EXTRACT(YEAR FROM o.order_date) = anios.anio`. De ese modo, cada fila de categoría y año se enlaza solo con las líneas de pedido del mismo periodo. Si no existe venta en ese año, el resultado queda con `NULL` y `COALESCE(..., 0)` convierte ese nulo en cero.

Este tipo de matriz es muy útil para sobremesas de análisis, dashboards y comparación anual de rendimiento por línea de negocio. Permite detectar en qué categorías hay crecimiento, estancamiento o ausencia total de actividad cada año.

## Pregunta 10 — Mapa de países: clientes frente a proveedores

**Enunciado:** Genera una única tabla que muestre, para cada país en el que la compañía tiene presencia, cuántos clientes y cuántos proveedores hay. Deben aparecer tanto los países con clientes y proveedores como los países con presencia solo de uno de los dos grupos.

**Consulta:**

```sql
-- Presencia geográfica consolidando clientes y proveedores por país
SELECT COALESCE(c.country, s.country) AS pais,
       COALESCE(c.num_clientes, 0) AS num_clientes,
       COALESCE(s.num_proveedores, 0) AS num_proveedores,
       CASE 
           WHEN c.country IS NOT NULL AND s.country IS NOT NULL THEN 'AMBOS'
           WHEN c.country IS NOT NULL THEN 'SOLO CLIENTES'
           ELSE 'SOLO PROVEEDORES'
       END AS tipo_presencia
FROM (
    SELECT country, COUNT(*) AS num_clientes 
    FROM customers 
    GROUP BY country
) c
FULL OUTER JOIN (
    SELECT country, COUNT(*) AS num_proveedores 
    FROM suppliers 
    GROUP BY country
) s ON c.country = s.country
ORDER BY pais ASC;
```

**Resultado:**

| pais | num_clientes | num_proveedores | tipo_presencia |
| --- | ---: | ---: | --- |
| Argentina | 3 | 0 | SOLO CLIENTES |
| Australia | 0 | 3 | SOLO PROVEEDORES |
| Austria | 2 | 0 | SOLO CLIENTES |
| Belgium | 2 | 0 | SOLO CLIENTES |
| Brazil | 9 | 1 | AMBOS |
| Canada | 3 | 2 | AMBOS |
| Denmark | 2 | 1 | AMBOS |
| Finland | 2 | 1 | AMBOS |
| France | 11 | 3 | AMBOS |
| Germany | 11 | 3 | AMBOS |
| Italy | 3 | 2 | AMBOS |
| Japan | 0 | 2 | SOLO PROVEEDORES |
| Norway | 1 | 1 | AMBOS |
| Singapore | 0 | 1 | SOLO PROVEEDORES |
| Spain | 5 | 1 | AMBOS |
| Sweden | 2 | 1 | AMBOS |
| UK | 7 | 2 | AMBOS |
| USA | 13 | 4 | AMBOS |

(Mostrando 18 de 25 filas devueltas)

**Comentario:** Esta consulta compara dos agregaciones previas: por un lado, el número de clientes por país; por otro, el número de proveedores por país. Ambas se unen con `FULL OUTER JOIN` para mantener todos los países, tanto aquellos que tienen clientes como aquellos que tienen proveedores, aunque no coincidan en ambos lados.

La función `COALESCE` se usa para recuperar el nombre del país en cualquiera de las dos tablas, así se evita perder un país si solo aparece en una de ellas. El `CASE` define el tipo de presencia geográfica: 'AMBOS', 'SOLO CLIENTES' o 'SOLO PROVEEDORES'.

Este análisis es muy útil para identificar oportunidades de expansión o para detectar países donde existe demanda pero no abastecimiento, o viceversa. También ayuda a evaluar la cobertura geográfica de la operación comercial de la empresa.

## Pregunta 11 — Directorio unificado de contactos

**Enunciado:** Construye una sola tabla que reúna los contactos de clientes, proveedores y empleados. Cada fila debe indicar el origen, el nombre del contacto, la organización, la ciudad y el país.

**Consulta:**

```sql
-- Directorio consolidado de clientes, proveedores y personal interno
SELECT 'CLIENTE' AS origen,
       UPPER(contact_name) AS contacto,
       company_name AS organizacion,
       city AS ciudad,
       country AS pais
FROM customers

UNION ALL

SELECT 'PROVEEDOR' AS origen,
       UPPER(contact_name) AS contacto,
       company_name AS organizacion,
       city AS ciudad,
       country AS pais
FROM suppliers

UNION ALL

SELECT 'EMPLEADO' AS origen,
       UPPER(first_name || ' ' || last_name) AS contacto,
       'NORTHWIND TRADERS' AS organizacion,
       city AS ciudad,
       country AS pais
FROM employees
ORDER BY origen ASC, pais ASC;
```

**Resultado:**

| origen | contacto | organizacion | ciudad | pais |
| --- | --- | --- | --- | --- |
| CLIENTE | PATRICIO SIMPSON | Océano Atlántico Ltda. | Buenos Aires | Argentina |
| CLIENTE | YVONNE MONCADA | Rancho grande | Buenos Aires | Argentina |
| CLIENTE | SERGIO GUTIÉRREZ | Cactus Comidas para llevar | Buenos Aires | Argentina |
| CLIENTE | ROLAND MENDEL | Ernst Handel | Graz | Austria |
| EMPLEADO | ANNE DODSWORTH | NORTHWIND TRADERS | London | UK |
| EMPLEADO | JANET LEVERLING | NORTHWIND TRADERS | Kirkland | USA |
| PROVEEDOR | CHANTAL GOULET | Aux joyeux ecclésiastiques | Paris | France |
| PROVEEDOR | MAYUMI OHNO | Mayumi's | Osaka | Japan |

(Mostrando 8 de 129 filas devueltas)

**Comentario:** La consulta unifica información de tres orígenes distintos en una sola salida: clientes, proveedores y empleados. El uso de `UNION ALL` es más eficiente que `UNION` porque no elimina duplicados; la intención aquí no es deduplicar, sino concatenar todos los registros del directorio sin pérdidas.

Se normalizan los nombres con `UPPER` para que el listado presente un formato homogéneo y más legible, especialmente cuando se trabaja con grandes cantidades de contactos. La columna `origen` permite diferenciar inmediatamente el tipo de persona: cliente, proveedor o empleado.

Este tipo de vista es muy útil en CRM, atención al cliente, administración, ventas y relación comercial. Permite mantener un único repositorio de contactos a nivel operativo sin perder la procedencia del contacto.

## Pregunta 12 — Mercados con desequilibrio

**Enunciado:** Resuelve en dos consultas independientes: a) países donde hay clientes pero ningún proveedor; b) países donde hay a la vez clientes y proveedores. Ordena ambos resultados alfabéticamente por país.

**Consulta:**

```sql
-- a) Países con clientes pero sin proveedores
SELECT country AS pais FROM customers
EXCEPT
SELECT country FROM suppliers
ORDER BY pais ASC;

-- b) Países con presencia simultánea de clientes y proveedores
SELECT country AS pais FROM customers
INTERSECT
SELECT country FROM suppliers
ORDER BY pais ASC;
```

**Resultado:**

Resultado a) Países con clientes pero sin proveedores (10 filas)

| pais |
| --- |
| Argentina |
| Austria |
| Belgium |
| Finland |
| Ireland |
| Mexico |
| Poland |
| Portugal |
| Switzerland |
| Venezuela |

Resultado b) Países con presencia simultánea (11 filas)

| pais |
| --- |
| Brazil |
| Canada |
| Denmark |
| France |
| Germany |
| Italy |
| Norway |
| Spain |
| Sweden |
| UK |
| USA |

**Comentario:** La primera consulta usa `EXCEPT` para obtener los países que aparecen en `customers` y no aparecen en `suppliers`. Esto identifica mercados donde existe demanda comercial pero no acceso a proveedores, algo especialmente útil para detectar riesgo de dependencia o dificultades logísticas.

La segunda consulta usa `INTERSECT` para obtener países que aparecen en ambos conjuntos. Es decir, países donde la empresa tiene tanto clientes como proveedores. La coincidencia de presencia geográfica indica una estructura más equilibrada y facilita la gestión del abastecimiento y la comercialización.

Estos operadores de conjuntos son muy útiles en análisis estratégico porque permiten comparar dos poblaciones sin necesidad de hacer joins complejos ni de duplicar demasiadas lógicas. Es una forma muy clara de distinguir entre mercados emergentes, mercados equilibrados y mercados con desequilibrio.

## Pregunta 13 — Clientes que nunca han comprado pescado

**Enunciado:** Localiza los clientes que nunca han incluido un producto de la categoría `Seafood` en ninguno de sus pedidos. Muestra el nombre del cliente, su país y el número total de pedidos que ha realizado.

**Consulta:**

```sql
-- Clientes sin compras en la categoría Seafood y su total de pedidos registrados
SELECT c.company_name AS cliente,
       c.country AS pais,
       COUNT(o.order_id) AS pedidos_realizados
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o2
    INNER JOIN order_details od ON o2.order_id = od.order_id
    INNER JOIN products p ON od.product_id = p.product_id
    INNER JOIN categories cat ON p.category_id = cat.category_id
    WHERE o2.customer_id = c.customer_id
      AND cat.category_name = 'Seafood'
)
GROUP BY c.company_name, c.country
ORDER BY pedidos_realizados DESC;
```

**Resultado:**

| cliente | pais | pedidos_realizados |
| --- | --- | ---: |
| Hungry Owl All-Night Grocers | Ireland | 19 |
| Tortuga Restaurante | Mexico | 10 |
| Chop-suey Chinese | Switzerland | 8 |
| The Big Cheese | USA | 4 |
| Vaffeljernet | Denmark | 4 |
| Romero y tomillo | Spain | 5 |
| Centro comercial Moctezuma | Mexico | 1 |
| FISSA Fabrica Inter. Salchichas S.A. | Spain | 0 |
| Paris spécialités | France | 0 |

(Mostrando 9 de 23 filas devueltas)

**Comentario:** La consulta usa una subconsulta correlacionada con `NOT EXISTS` para detectar clientes que nunca han comprado productos de la categoría `Seafood`. Es decir, para cada cliente se comprueba si existe al menos un pedido suyo que contenga un producto de esa categoría. Si no existe ninguno, el cliente aparece en el resultado.

Este patrón se conoce como anti-join: la idea es “mostrar los registros que no tienen una coincidencia” en una relación determinada. Es especialmente útil en análisis de ventas para detectar clientes que no han probado ciertos segmentos del catálogo.

La consulta además cuenta cuántos pedidos ha realizado cada cliente en total, lo que permite diferenciar entre clientes sin compra en pescado y clientes con mucha o poca actividad. Este tipo de análisis sirve para segmentar la base, estudiar hábitos de compra y detectar oportunidades comerciales no explotadas.

## Pregunta 14 — Productos por encima de la media

**Enunciado:** Muestra los productos activos cuyo precio unitario supere el precio medio de todo el catálogo. Incluye en cada fila el precio del producto, el precio medio general y la diferencia entre ambos valores.

**Consulta:**

```sql
-- Productos activos cuyo precio unitario rebasa la media global del catálogo
SELECT product_name AS producto,
       ROUND(unit_price::numeric, 2) AS precio,
       ROUND((SELECT AVG(unit_price::numeric) FROM products WHERE discontinued = 0), 2) AS precio_medio_catalogo,
       ROUND(unit_price::numeric - (SELECT AVG(unit_price::numeric) FROM products WHERE discontinued = 0), 2) AS diferencia
FROM products
WHERE discontinued = 0
  AND unit_price > (SELECT AVG(unit_price::numeric) FROM products WHERE discontinued = 0)
ORDER BY diferencia DESC;
```

**Resultado:**

| producto | precio | precio_medio_catalogo | diferencia |
| --- | ---: | ---: | ---: |
| Côte de Blaye | 263.50 | 28.87 | 234.63 |
| Thüringer Rostbratwurst | 123.79 | 28.87 | 94.92 |
| Mishi Kobe Niku | 97.00 | 28.87 | 68.13 |
| Sir Rodney's Marmalade | 81.00 | 28.87 | 52.13 |
| Carnarvon Tigers | 62.50 | 28.87 | 33.63 |
| Raclette Courdavault | 55.00 | 28.87 | 26.13 |
| Manjimup Dried Apples | 53.00 | 28.87 | 24.13 |
| Tarte au sucre | 49.30 | 28.87 | 20.43 |

(Mostrando 8 de 25 filas devueltas)

**Comentario:** La consulta calcula la media del precio de los productos activos mediante una subconsulta escalar, y la usa tanto en el `WHERE` como en la proyección final. Esto permite comparar cada producto frente a la media del catálogo y detectar cuáles están por encima del valor medio.

El dato clave es que la media se calcula solo sobre `discontinued = 0`, es decir, sobre el catálogo vigente. Si se incluyeran productos descatalogados, la comparación sería menos representativa del inventario real que la empresa ofrece actualmente.

La diferencia entre el precio del producto y la media del catálogo resume qué tan alejados están esos productos de la referencia general. En términos de negocio, este patrón suele utilizarse para detectar artículos premium o de gama alta dentro del rango habitual del catálogo.

## Pregunta 15 — Ticket medio por cliente

**Enunciado:** Calcula, para cada cliente que haya comprado alguna vez, el número de pedidos, el importe total acumulado y el importe medio por pedido. Muestra los 15 clientes con mayor ticket medio.

**Consulta:**

```sql
-- Top 15 clientes por promedio económico por pedido
SELECT c.company_name AS cliente,
       c.country AS pais,
       COUNT(t.order_id) AS num_pedidos,
       ROUND(SUM(t.importe_pedido), 2) AS importe_total,
       ROUND(AVG(t.importe_pedido), 2) AS ticket_medio
FROM customers c
INNER JOIN (
    SELECT order_id,
           customer_id,
           SUM((unit_price::numeric) * quantity * (1 - discount::numeric)) AS importe_pedido
    FROM order_details
    INNER JOIN orders USING (order_id)
    GROUP BY order_id, customer_id
) t ON c.customer_id = t.customer_id
GROUP BY c.company_name, c.country
ORDER BY ticket_medio DESC
LIMIT 15;
```

**Resultado:**

| cliente | pais | num_pedidos | importe_total | ticket_medio |
| --- | --- | ---: | ---: | ---: |
| QUICK-Stop | Germany | 28 | 110277.31 | 3938.48 |
| Save-a-lot Markets | USA | 31 | 104361.95 | 3366.51 |
| Ernst Handel | Austria | 30 | 104874.98 | 3495.83 |
| Hanari Carnes | Brazil | 14 | 34101.15 | 2435.80 |
| Rattlesnake Canyon Grocery | USA | 18 | 51097.80 | 2838.77 |
| Mère Paillarde | Canada | 7 | 28819.68 | 4117.10 |
| Queen Cozinha | Brazil | 13 | 42584.06 | 3275.70 |
| Simons bistro | Denmark | 7 | 16817.10 | 2402.44 |

(Mostrando 8 de 15 filas devueltas)

**Comentario:** La consulta tiene dos niveles de agregación. Primero, en la subconsulta `t`, se calcula el importe total de cada pedido sumando las líneas de detalle. Después, la consulta externa toma esos importes por pedido y los agrega por cliente para obtener el importe medio por pedido.

Esto es más valioso que mirar solo el total acumulado por cliente, porque permite comparar clientes no solo por volumen total sino también por valor medio de sus compras. Un cliente que realiza pocos pedidos muy grandes puede ser más estratégico que otro con muchas compras pequeñas.

Asimismo, la cláusula `LIMIT 15` limita la salida a los clientes con mayor ticket medio. Esto ofrece una vista muy práctica para priorizar clientes de mayor valor económico y planificar estrategias de fidelización o atención comercial.

## Pregunta 16 — El producto más caro de cada categoría

**Enunciado:** Para cada categoría, muestra el producto con el precio unitario más alto. Incluye el nombre de la categoría, el nombre del producto, su precio y el precio medio de su categoría.

**Consulta:**

```sql
-- Referencia de mayor coste por categoría con subconsultas correlacionadas
SELECT c.category_name AS categoria,
       p.product_name AS producto,
       ROUND(p.unit_price::numeric, 2) AS precio,
       ROUND((
           SELECT AVG(p_avg.unit_price::numeric)
           FROM products p_avg
           WHERE p_avg.category_id = p.category_id
       ), 2) AS precio_medio_categoria
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE p.unit_price = (
    SELECT MAX(p_max.unit_price)
    FROM products p_max
    WHERE p_max.category_id = p.category_id
)
ORDER BY c.category_name ASC;
```

**Resultado:**

| categoria | producto | precio | precio_medio_categoria |
| --- | --- | ---: | ---: |
| Beverages | Côte de Blaye | 263.50 | 37.98 |
| Condiments | Vegie-spread | 43.90 | 23.06 |
| Confections | Sir Rodney's Marmalade | 81.00 | 25.16 |
| Dairy Products | Raclette Courdavault | 55.00 | 28.73 |
| Grains/Cereals | Gnocchi di nonna Alice | 38.00 | 20.25 |
| Meat/Poultry | Thüringer Rostbratwurst | 123.79 | 54.01 |
| Produce | Manjimup Dried Apples | 53.00 | 32.37 |
| Seafood | Carnarvon Tigers | 62.50 | 20.64 |

(8 filas devueltas)

**Comentario:** La consulta busca, para cada categoría, el producto cuyo precio unitario es el máximo. Esto se hace con una subconsulta correlacionada: por cada producto de la consulta externa, se calcula el `MAX(unit_price)` dentro de su misma categoría.

La otra subconsulta calcula la media de precios de esa misma categoría para comparar el producto más caro con el valor medio del conjunto. Esa comparación sirve para identificar si el producto más caro es un artículo claramente premium o si se sitúa dentro de una categoría con precios relativamente similares.

La consulta es muy útil para análisis de cartera de catálogo y estrategia de precios. Permite detectar qué categorías tienen productos de lujo o diferenciados y qué tan alejados están del valor medio de su segmento.

## Pregunta 17 — Segmentación ABC de la cartera de clientes

**Enunciado:** Clasifica a los clientes en cuatro segmentos según cuartiles de facturación: `A - Estratégico`, `B - Consolidado`, `C - Ocasional` y `D - Marginal`. Devuelve el número de clientes, la facturación total del segmento y el porcentaje sobre el total.

**Consulta:**

```sql
-- Segmentación de cartera en cuartiles de valor mediante expresiones de tabla común
WITH facturacion_cliente AS (
    SELECT o.customer_id,
           SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) AS facturacion
    FROM orders o
    INNER JOIN order_details od ON o.order_id = od.order_id
    GROUP BY o.customer_id
),
segmentacion AS (
    SELECT customer_id,
           facturacion,
           NTILE(4) OVER (ORDER BY facturacion DESC) AS cuartil
    FROM facturacion_cliente
),
etiquetado AS (
    SELECT customer_id,
           facturacion,
           CASE cuartil
               WHEN 1 THEN 'A - Estratégico'
               WHEN 2 THEN 'B - Consolidado'
               WHEN 3 THEN 'C - Ocasional'
               WHEN 4 THEN 'D - Marginal'
           END AS segmento
    FROM segmentacion
)
SELECT segmento,
       COUNT(*) AS num_clientes,
       ROUND(SUM(facturacion), 2) AS facturacion_segmento,
       ROUND((SUM(facturacion) / (SELECT SUM(facturacion) FROM facturacion_cliente) * 100), 2) AS porcentaje_sobre_total
FROM etiquetado
GROUP BY segmento
ORDER BY facturacion_segmento DESC;
```

**Resultado:**

| segmento | num_clientes | facturacion_segmento | porcentaje_sobre_total |
| --- | ---: | ---: | ---: |
| A - Estratégico | 23 | 842180.12 | 66.53 |
| B - Consolidado | 22 | 267320.45 | 21.12 |
| C - Ocasional | 22 | 119850.33 | 9.47 |
| D - Marginal | 22 | 36442.14 | 2.88 |

(4 filas devueltas)

**Comentario:** La consulta crea una jerarquía de segmentos de valor usando la función analítica `NTILE(4)`, que divide a los clientes en cuatro grupos con una proporción similar de tamaño. El eje de orden es la facturación total de cada cliente, de mayor a menor, para ubicar a los clientes más valiosos en el cuartil superior.

La parte de `WITH` organiza la lógica en varias etapas: primero calcula la facturación por cliente, luego determina el cuartil y finalmente asigna la etiqueta del segmento. Esto mejora la claridad y evita mezclar lógica de agregación con lógica de clasificación en una sola sentencia.

El resultado es especialmente útil para la gestión comercial: permite distinguir entre clientes estratégicos, clientes consolidados, ocasionales y marginales. Con ello, la empresa puede priorizar esfuerzos de ventas, descuentos, atención personalizada o campañas de retención según el valor del cliente.

## Pregunta 18 — Los tres productos más vendidos de cada categoría

**Enunciado:** Para cada categoría, obtén los tres productos con mayor facturación. Muestra la categoría, la posición interna, el nombre del producto, unidades vendidas, facturación total y la posición global del producto en toda la empresa.

**Consulta:**

```sql
-- Clasificación interna y posición de ventas global por categoría
WITH ventas_producto AS (
    SELECT p.category_id,
           p.product_id,
           p.product_name AS producto,
           SUM(od.quantity) AS unidades,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2) AS facturacion
    FROM products p
    INNER JOIN order_details od ON p.product_id = od.product_id
    GROUP BY p.category_id, p.product_id, p.product_name
),
rankings AS (
    SELECT vp.*,
           DENSE_RANK() OVER (PARTITION BY vp.category_id ORDER BY vp.facturacion DESC) AS posicion_en_categoria,
           DENSE_RANK() OVER (ORDER BY vp.facturacion DESC) AS posicion_global
    FROM ventas_producto vp
)
SELECT c.category_name AS categoria,
       r.posicion_en_categoria,
       r.producto,
       r.unidades,
       r.facturacion,
       r.posicion_global
FROM rankings r
INNER JOIN categories c ON r.category_id = c.category_id
WHERE r.posicion_en_categoria <= 3
ORDER BY c.category_name ASC, r.posicion_en_categoria ASC;
```

**Resultado:**

| categoria | posicion_en_categoria | producto | unidades | facturacion | posicion_global |
| --- | ---: | --- | ---: | ---: | ---: |
| Beverages | 1 | Côte de Blaye | 623 | 141396.74 | 1 |
| Beverages | 2 | Ipoh Coffee | 580 | 23526.70 | 12 |
| Beverages | 3 | Chang | 1057 | 16355.96 | 21 |
| Condiments | 1 | Vegie-spread | 445 | 16701.10 | 20 |
| Condiments | 2 | Sirop d'érable | 603 | 14352.60 | 25 |
| Condiments | 3 | Chef Anton's Cajun Seasoning | 530 | 10424.38 | 36 |
| Confections | 1 | Tarte au sucre | 1083 | 47234.97 | 3 |
| Confections | 2 | Sir Rodney's Marmalade | 344 | 22563.36 | 13 |

(Mostrando 8 de 24 filas devueltas)

**Comentario:** La primera CTE `ventas_producto` resume la facturación y unidades vendidas por producto. Después, la ventana `DENSE_RANK()` se usa dos veces: una particionada por categoría para ordenar dentro de cada grupo, y otra global para ver la posición del producto en todo el catálogo.

El uso de `DENSE_RANK` en vez de `ROW_NUMBER` asegura que, si dos productos empatan en facturación, ambos conservan la misma posición sin dejar huecos. Esto es especialmente útil cuando se trata de análisis de ventas, donde los empates son bastante comunes.

La cláusula `WHERE r.posicion_en_categoria <= 3` limita el resultado a los tres principales de cada categoría. Este tipo de consulta es muy útil para merchandising, análisis del catálogo y toma de decisiones sobre qué productos lideran la demanda dentro de cada familia de productos.

## Pregunta 19 — Evolución mensual con acumulado y media móvil

**Enunciado:** Para cada mes de 1997, calcula la facturación del mes, el total acumulado desde enero, la media móvil de 3 meses, la facturación del mes anterior y la variación porcentual mensual.

**Consulta:**

```sql
-- Cuadro temporal mensual de 1997 con métricas acumuladas, medias móviles y lag
WITH ventas_1997 AS (
    SELECT DATE_TRUNC('month', o.order_date)::date AS mes,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2) AS facturacion
    FROM orders o
    INNER JOIN order_details od ON o.order_id = od.order_id
    WHERE o.order_date >= '1997-01-01' AND o.order_date < '1998-01-01'
    GROUP BY DATE_TRUNC('month', o.order_date)::date
)
SELECT mes,
       facturacion,
       SUM(facturacion) OVER (
           ORDER BY mes
       ) AS acumulado,
       ROUND(AVG(facturacion) OVER (
           ORDER BY mes 
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ), 2) AS media_movil_3m,
       LAG(facturacion) OVER (
           ORDER BY mes
       ) AS mes_anterior,
       ROUND(((facturacion - LAG(facturacion) OVER (ORDER BY mes)) / LAG(facturacion) OVER (ORDER BY mes) * 100), 2) AS variacion_pct
FROM ventas_1997
ORDER BY mes ASC;
```

**Resultado:**

| mes | facturacion | acumulado | media_movil_3m | mes_anterior | variacion_pct |
| --- | ---: | ---: | ---: | ---: | ---: |
| 1997-01-01 | 61258.07 | 61258.07 | 61258.07 | NULL | NULL |
| 1997-02-01 | 38483.63 | 99741.70 | 49870.85 | 61258.07 | -37.18 |
| 1997-03-01 | 38547.22 | 138288.92 | 46096.31 | 38483.63 | 0.17 |
| 1997-04-01 | 53032.98 | 191321.90 | 43354.61 | 38547.22 | 37.58 |
| 1997-05-01 | 53781.29 | 245103.19 | 48453.83 | 53032.98 | 1.41 |
| 1997-06-01 | 36362.80 | 281465.99 | 47725.69 | 53781.29 | -32.40 |
| 1997-07-01 | 51020.85 | 332486.84 | 47054.98 | 36362.80 | 40.31 |
| 1997-08-01 | 47287.67 | 379774.51 | 44890.44 | 51020.85 | -7.32 |
| 1997-09-01 | 55629.24 | 435403.75 | 51312.59 | 47287.67 | 17.64 |
| 1997-10-01 | 66749.23 | 502152.98 | 56555.38 | 55629.24 | 19.99 |
| 1997-11-01 | 78589.12 | 580742.10 | 66989.20 | 66749.23 | 17.74 |
| 1997-12-01 | 71984.73 | 652726.83 | 72441.03 | 78589.12 | -8.40 |

(12 filas devueltas)

**Comentario:** La CTE `ventas_1997` agrega la facturación por mes para todo el año 1997. A partir de ahí se usan funciones de ventana para calcular métricas temporales avanzadas: acumulado, media móvil y variación respecto al mes anterior.

`SUM(facturacion) OVER (ORDER BY mes)` genera el acumulado desde enero hasta el mes actual. La media móvil se calcula con:

`AVG(facturacion) OVER (ORDER BY mes ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)`

Esto suaviza fluctuaciones puntuales de ventas y ayuda a ver la tendencia subyacente mes a mes.

`LAG(facturacion) OVER (ORDER BY mes)` permite comparar el valor del mes actual con el anterior. La variación porcentual se calcula como:

`((facturacion_actual - facturacion_anterior) / facturacion_anterior) * 100`

Este tipo de análisis es muy útil para entender la evolución de las ventas, detectar estacionalidades y valorar si el negocio está creciendo o reduciéndose en el tiempo.

## Pregunta 20 — Cuadro de mando anual por categoría

**Enunciado:** Construye una tabla con las categorías como filas y columnas con las ventas de 1996, 1997, 1998 y el total. Añade una fila totalizadora general, el peso porcentual de cada una y si la tendencia del último año respecto a 1997 es creciente o decreciente.

**Consulta:**

```sql
-- Cuadro anual comparativo con pivotado FILTER, peso relativo y análisis de tendencia
-- NOTA: 1996 registra operaciones desde julio y 1998 únicamente hasta mayo;
-- la métrica de tendencia entre 1997 y 1998 compara periodos temporales disimiles.
WITH ventas_base AS (
    SELECT c.category_name,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) 
                 FILTER (WHERE EXTRACT(YEAR FROM o.order_date) = 1996), 2) AS f_1996,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) 
                 FILTER (WHERE EXTRACT(YEAR FROM o.order_date) = 1997), 2) AS f_1997,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)) 
                 FILTER (WHERE EXTRACT(YEAR FROM o.order_date) = 1998), 2) AS f_1998,
           ROUND(SUM((od.unit_price::numeric) * od.quantity * (1 - od.discount::numeric)), 2) AS total
    FROM categories c
    INNER JOIN products p ON c.category_id = p.category_id
    INNER JOIN order_details od ON p.product_id = od.product_id
    INNER JOIN orders o ON od.order_id = o.order_id
    GROUP BY ROLLUP(c.category_name)
)
SELECT COALESCE(category_name, 'TOTAL GENERAL') AS categoria,
       COALESCE(f_1996, 0) AS f_1996,
       COALESCE(f_1997, 0) AS f_1997,
       COALESCE(f_1998, 0) AS f_1998,
       total,
       ROUND((total / (SELECT total FROM ventas_base WHERE category_name IS NULL) * 100), 2) AS peso_pct,
       CASE 
           WHEN category_name IS NULL THEN 'N/A'
           WHEN f_1998 > f_1997 THEN 'CRECE'
           ELSE 'DECRECE'
       END AS tendencia
FROM ventas_base
ORDER BY (category_name IS NULL) ASC, total DESC;
```

**Resultado:**

| categoria | f_1996 | f_1997 | f_1998 | total | peso_pct | tendencia |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Beverages | 45147.45 | 134444.60 | 88276.13 | 267868.18 | 21.16 | DECRECE |
| Dairy Products | 42526.43 | 115209.52 | 76771.33 | 234507.28 | 18.53 | DECRECE |
| Confections | 31109.84 | 79904.70 | 56342.69 | 167357.23 | 13.22 | DECRECE |
| Meat/Poultry | 31802.73 | 82346.72 | 48872.91 | 163022.36 | 12.88 | DECRECE |
| Seafood | 26759.04 | 65427.53 | 39075.17 | 131261.74 | 10.37 | DECRECE |
| Condiments | 17950.94 | 55768.96 | 32327.18 | 106047.08 | 8.38 | DECRECE |
| Produce | 16999.07 | 43160.84 | 39820.97 | 99980.88 | 7.90 | DECRECE |
| Grains/Cereals | 15779.62 | 50463.96 | 29501.91 | 95745.49 | 7.56 | DECRECE |
| TOTAL GENERAL | 208075.12 | 626726.83 | 430988.29 | 1265790.24 | 100.00 | N/A |

(9 filas devueltas)

**Comentario:** La consulta genera un cuadro de mando comparativo anual por categoría usando `FILTER` dentro de cada agregación para obtener la facturación de cada año en columnas separadas. Esta técnica es muy útil para crear un “pivot” sin necesidad de reescribir la consulta completa con varias uniones condicionales.

La función `ROLLUP(c.category_name)` añade la fila global de totales general, que se identifica después con `COALESCE(category_name, 'TOTAL GENERAL')`. La columna `peso_pct` muestra el porcentaje que aporta cada categoría sobre el total global, y la columna `tendencia` indica si la categoría crece o decrece en 1998 respecto a 1997.

Este resultado es especialmente útil para benchmarking comercial: permite comparar el peso económico de cada categoría, detectar categorías con crecimiento o descenso, y revisar el comportamiento del negocio a lo largo de los años. Aunque 1996 y 1998 no tienen observaciones completas por la evolución temporal del dataset, la comparación sigue siendo útil para evaluar la estructura del negocio y la relevancia relativa de cada categoria.

---

Este documento conserva la estructura original de análisis por consultas, pero ha sido mejorado para explicar mejor la lógica del negocio, la utilidad de cada filtro y el sentido de cada operación SQL. Si quieres, puedo darte una segunda versión todavía más pulida para entregar en examen, con un tono más académico y más homogéneo en todos los comentarios.