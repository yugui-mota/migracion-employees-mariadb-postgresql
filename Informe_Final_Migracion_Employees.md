# Entregable final: migración de MariaDB a PostgreSQL

| Datos académicos | Información |
|---|---|
| Estudiante | Almendras Davalos Juan Junior |
| Docente | Lopez Leaño Jared |
| Asignatura | Tecnología de Base de Datos I |
| Unidad | Bloque 3 - Migración de un sistema informático a otro SGBD |
| Tema | Migración de tablas, vistas y consultas de verificación |


## Introducción

Este informe documenta la migración de la base `employees` desde MariaDB a
PostgreSQL 18, utilizando `pdb_employees` como base de destino y `employees`
como esquema. Se retoman las evidencias de la migración de tablas realizada
con pgloader y se incorporarán la migración de vistas, las verificaciones
de integridad, el respaldo y la entrega mediante GitHub.

**Estado:** documento en elaboración. Las acciones pendientes no se presentan
como ejecutadas. Los comandos y sus salidas se documentan en texto.

## 1. Migración de tablas mediante pgloader

### 1.1 Origen y destino

El origen contiene seis tablas: `departments`, `dept_emp`, `dept_manager`,
`employees`, `salaries` y `titles`. También contiene las vistas
`dept_emp_latest_date` y `current_dept_emp`, tratadas en el punto 2.

Se creó la base de destino desde la terminal:

```bash
psql -h localhost -U juan -d pdb_juan -c "CREATE DATABASE pdb_employees;"
```

Salida registrada:

```text
Contraseña para usuario juan:
CREATE DATABASE
```

Conexión posterior:

```bash
psql -h localhost -U juan -d pdb_employees
```

La sesión mostró el indicador `pdb_employees=#`, confirmando la conexión.

### 1.2 Exportación de la estructura desde MariaDB

Comando utilizado:

```bash
mariadb-dump -h 127.0.0.1 -u root -p --no-data employees > employees-estructura.sql
```

La opción `--no-data` exporta las definiciones de los objetos, sin los registros.
`-p` solicita la contraseña y `>` guarda la salida en `employees-estructura.sql`;
si el archivo existe, reemplaza su contenido.

Se comprobó la existencia del archivo mediante:

```bash
ls -lsh employees-estructura.sql
```

Salida registrada:

```text
8,0K -rw-r--r-- 1 juan juan 7,9K sep 23 12:25 employees-estructura.sql
```

Este archivo documenta la exportación de estructura. No se utilizó como entrada
de pgloader: la migración de tablas y registros se realizó mediante conexión
directa a MariaDB, con PostgreSQL como destino.

### 1.3 Configuración y adaptación de tipos

Configuración documentada de `migracion.load`, con las contraseñas sustituidas
por marcadores para la entrega:

```lisp
LOAD DATABASE
    FROM mysql://root:TU_CLAVE@localhost:3306/employees
    INTO postgresql://juan:TU_CLAVE@localhost:5432/pdb_employees

WITH include drop,
     create tables,
     create indexes,
     reset sequences,
     workers = 8, concurrency = 2,
     batch rows = 10000

SET maintenance_work_mem to '256MB',
    work_mem to '32MB'

CAST type datetime to timestamp drop default drop not null
     using zero-dates-to-null
;
```

Se eliminó la regla `type enum to text` para permitir la conversión automática
del enumerado de MariaDB a un tipo PostgreSQL. La regla para `datetime` se
conservó, pero no afecta a las seis tablas examinadas, que utilizan `date`.
`include drop` permite eliminar y recrear las tablas de destino; esta
configuración documenta la ejecución realizada y no debe repetirse sin
revisar los objetos existentes en el destino.

Las verificaciones posteriores mostraron los enteros `int(11)` como `bigint`,
las longitudes de caracteres conservadas y `gender` como `employees_gender`.
Ninguna de las tablas de origen examinadas declara `AUTO_INCREMENT`.

### 1.4 Importación mediante pgloader y resultado

Comando ejecutado desde `~/tecBD1`:

```bash
pgloader migracion.load
```

Extracto de la salida registrada, con formato resumido:

```text
pgloader version "3.6.10~devel"
Parsing commands from file #P"/home/juan/tecBD1/migracion.load"
Origen: mysql://root@localhost:3306/employees
Destino: pgsql://juan@localhost:5432/pdb_employees

Operación / tabla          errors     rows
Create SQL Types               0        1
employees.salaries             0  2844047
employees.titles               0   443308
employees.dept_emp             0   331603
employees.dept_manager         0       24
employees.employees            0   300024
employees.departments          0        9
Create Indexes                 0        9
Primary Keys                   0        6
Create Foreign Keys            0        6

Total import time: 3919015 rows; 134.9 MB; 30.720s
```

Pgloader no informó errores en el resumen recibido. Se cargaron 3919015 filas
en seis tablas. Los conteos independientes se documentan en el apartado 1.6.

### 1.5 Verificación inicial del destino

```text
\dt employees.*
```

Salida registrada en `pdb_employees`:

```text
  Esquema  |    Nombre    | Tipo  | Dueño
-----------+--------------+-------+-------
 employees | departments  | tabla | juan
 employees | dept_emp     | tabla | juan
 employees | dept_manager | tabla | juan
 employees | employees    | tabla | juan
 employees | salaries     | tabla | juan
 employees | titles       | tabla | juan
(6 filas)
```

La descripción `\d employees.employees` confirmó `gender` como
`employees_gender` con `NOT NULL`. Se comprobaron las etiquetas del tipo:

```sql
SELECT t.typname AS tipo, e.enumlabel AS valor
FROM pg_type AS t
JOIN pg_namespace AS n ON n.oid = t.typnamespace
JOIN pg_enum AS e ON e.enumtypid = t.oid
WHERE n.nspname = 'employees'
  AND t.typname = 'employees_gender'
ORDER BY e.enumsortorder;
```

```text
       tipo       | valor
------------------+-------
 employees_gender | M
 employees_gender | F
(2 filas)
```

### 1.6 Conteo de filas en MariaDB y PostgreSQL

Los conteos de las seis tablas se ejecutaron anteriormente en ambos motores.
Para MariaDB se seleccionó `employees`; en PostgreSQL la sesión estaba en
`pdb_employees`, con las tablas de `employees` visibles. Consulta ejecutada:

```sql
SELECT 'departments' AS tabla, COUNT(*) AS registros FROM departments
UNION ALL
SELECT 'dept_emp', COUNT(*) FROM dept_emp
UNION ALL
SELECT 'dept_manager', COUNT(*) FROM dept_manager
UNION ALL
SELECT 'employees', COUNT(*) FROM employees
UNION ALL
SELECT 'salaries', COUNT(*) FROM salaries
UNION ALL
SELECT 'titles', COUNT(*) FROM titles
ORDER BY tabla;
```

Salida de MariaDB:

```text
tabla        | registros
departments  | 9
dept_emp     | 331603
dept_manager | 24
employees    | 300024
salaries     | 2844047
titles       | 443308
6 rows in set (2,295 sec)
```

Salida de PostgreSQL:

```text
tabla        | registros
departments  | 9
dept_emp     | 331603
dept_manager | 24
employees    | 300024
salaries     | 2844047
titles       | 443308
(6 filas)
```

Se simplificaron los bordes y espacios de ambas salidas sin alterar los valores.

| Tabla | MariaDB | PostgreSQL | Diferencia |
|---|---:|---:|---:|
| departments | 9 | 9 | 0 |
| dept_emp | 331603 | 331603 | 0 |
| dept_manager | 24 | 24 | 0 |
| employees | 300024 | 300024 | 0 |
| salaries | 2844047 | 2844047 | 0 |
| titles | 443308 | 443308 | 0 |
| Total | 3919015 | 3919015 | 0 |

Los conteos coinciden tabla por tabla. Esta evidencia confirma igualdad de
cantidades en el momento de las consultas, no igualdad de todos los valores.

## 2. Migración de vistas

### 2.1 Extracción desde MariaDB

Comandos ya ejecutados:

```sql
SHOW CREATE VIEW dept_emp_latest_date\G
SHOW CREATE VIEW current_dept_emp\G
```

Definiciones originales extraídas del campo `Create View` de las salidas:

```sql
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER
VIEW `dept_emp_latest_date` AS
SELECT `dept_emp`.`emp_no` AS `emp_no`,
       MAX(`dept_emp`.`from_date`) AS `from_date`,
       MAX(`dept_emp`.`to_date`) AS `to_date`
FROM `dept_emp`
GROUP BY `dept_emp`.`emp_no`;

CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%` SQL SECURITY DEFINER
VIEW `current_dept_emp` AS
SELECT `l`.`emp_no` AS `emp_no`, `d`.`dept_no` AS `dept_no`,
       `l`.`from_date` AS `from_date`, `l`.`to_date` AS `to_date`
FROM (`dept_emp` `d` JOIN `dept_emp_latest_date` `l`
  ON (`d`.`emp_no` = `l`.`emp_no`
  AND `d`.`from_date` = `l`.`from_date`
  AND `l`.`to_date` = `d`.`to_date`));
```

Ambas salidas indicaron `character_set_client: utf8mb3`,
`collation_connection: utf8mb3_uca1400_ai_ci` y
`1 row in set (0,001 sec)`. Se reorganizaron saltos de línea para su lectura.

### 2.2 Adaptación a PostgreSQL

Para `dept_emp_latest_date` se retiraron las cláusulas propias de MariaDB
`ALGORITHM=UNDEFINED`, `DEFINER` y `SQL SECURITY DEFINER`, así como las comillas
invertidas. Se calificaron los objetos con el esquema `employees`.
Se conservaron `MAX` y `GROUP BY`, compatibles con PostgreSQL.
La adaptación conserva la lógica de consulta; no reproduce automáticamente
la configuración de usuarios y permisos de MariaDB.

La vista calcula por separado las fechas máximas de inicio y fin por empleado;
no garantiza que ambas fechas procedan de una misma fila de origen.
Se creó primero esta vista porque `current_dept_emp` depende de ella.

Para `current_dept_emp` se retiraron las mismas opciones específicas de MariaDB
y las comillas invertidas. Se especificó el esquema `employees` en ambas
referencias y se conservaron las tres condiciones del `JOIN`: coincidencia
del empleado y de las fechas de inicio y fin.

### 2.3 Creación y pruebas

#### 2.3.1 Vista dept_emp_latest_date

Comando adaptado y ejecutado en `pdb_employees`:

```sql
CREATE VIEW employees.dept_emp_latest_date AS
SELECT emp_no,
       MAX(from_date) AS from_date,
       MAX(to_date) AS to_date
FROM employees.dept_emp
GROUP BY emp_no;
```

Salida registrada:

```text
CREATE VIEW
```

Consulta de prueba ejecutada:

```sql
SELECT *
FROM employees.dept_emp_latest_date
ORDER BY emp_no
LIMIT 10;
```

Salida registrada:

```text
 emp_no | from_date  |  to_date
--------+------------+------------
  10001 | 1986-06-26 | 9999-01-01
  10002 | 1996-08-03 | 9999-01-01
  10003 | 1995-12-03 | 9999-01-01
  10004 | 1986-12-01 | 9999-01-01
  10005 | 1989-09-12 | 9999-01-01
  10006 | 1990-08-05 | 9999-01-01
  10007 | 1989-02-10 | 9999-01-01
  10008 | 1998-03-11 | 2000-07-31
  10009 | 1985-02-18 | 9999-01-01
  10010 | 2000-06-26 | 9999-01-01
(10 filas)
```

La respuesta `CREATE VIEW` confirma la creación. La consulta devolvió las diez
filas solicitadas sin errores reportados. Esta prueba demuestra que la vista
puede consultarse; la comparación de la muestra con MariaDB se documenta
en el apartado 2.4.

#### 2.3.2 Vista current_dept_emp

Comando adaptado y ejecutado en `pdb_employees`:

```sql
CREATE VIEW employees.current_dept_emp AS
SELECT
    l.emp_no,
    d.dept_no,
    l.from_date,
    l.to_date
FROM employees.dept_emp AS d
JOIN employees.dept_emp_latest_date AS l
    ON d.emp_no = l.emp_no
   AND d.from_date = l.from_date
   AND d.to_date = l.to_date;
```

Salida registrada:

```text
CREATE VIEW
```

Consulta de prueba ejecutada:

```sql
SELECT *
FROM employees.current_dept_emp
ORDER BY emp_no, dept_no
LIMIT 10;
```

Salida registrada:

```text
 emp_no | dept_no | from_date  |  to_date
--------+---------+------------+------------
  10001 | d005    | 1986-06-26 | 9999-01-01
  10002 | d007    | 1996-08-03 | 9999-01-01
  10003 | d004    | 1995-12-03 | 9999-01-01
  10004 | d004    | 1986-12-01 | 9999-01-01
  10005 | d003    | 1989-09-12 | 9999-01-01
  10006 | d005    | 1990-08-05 | 9999-01-01
  10007 | d008    | 1989-02-10 | 9999-01-01
  10008 | d005    | 1998-03-11 | 2000-07-31
  10009 | d006    | 1985-02-18 | 9999-01-01
  10010 | d006    | 2000-06-26 | 9999-01-01
(10 filas)
```

La vista se creó y devolvió las diez filas solicitadas. Añade el departamento
de la asignación que coincide con las fechas calculadas en la primera vista.
A pesar de su nombre, no filtra por la fecha actual: incluye, por ejemplo,
la asignación del empleado `10008`, finalizada el `2000-07-31`.
Se conserva así la lógica original de MariaDB.

### 2.4 Comparación de muestras con MariaDB

Consultas ejecutadas en MariaDB:

```sql
USE employees;

SELECT *
FROM dept_emp_latest_date
ORDER BY emp_no
LIMIT 10;

SELECT *
FROM current_dept_emp
ORDER BY emp_no, dept_no
LIMIT 10;
```

Salida de `dept_emp_latest_date`:

```text
+--------+------------+------------+
| emp_no | from_date  | to_date    |
+--------+------------+------------+
|  10001 | 1986-06-26 | 9999-01-01 |
|  10002 | 1996-08-03 | 9999-01-01 |
|  10003 | 1995-12-03 | 9999-01-01 |
|  10004 | 1986-12-01 | 9999-01-01 |
|  10005 | 1989-09-12 | 9999-01-01 |
|  10006 | 1990-08-05 | 9999-01-01 |
|  10007 | 1989-02-10 | 9999-01-01 |
|  10008 | 1998-03-11 | 2000-07-31 |
|  10009 | 1985-02-18 | 9999-01-01 |
|  10010 | 2000-06-26 | 9999-01-01 |
+--------+------------+------------+
10 rows in set (0,687 sec)
```

Salida de `current_dept_emp`:

```text
+--------+---------+------------+------------+
| emp_no | dept_no | from_date  | to_date    |
+--------+---------+------------+------------+
|  10001 | d005    | 1986-06-26 | 9999-01-01 |
|  10002 | d007    | 1996-08-03 | 9999-01-01 |
|  10003 | d004    | 1995-12-03 | 9999-01-01 |
|  10004 | d004    | 1986-12-01 | 9999-01-01 |
|  10005 | d003    | 1989-09-12 | 9999-01-01 |
|  10006 | d005    | 1990-08-05 | 9999-01-01 |
|  10007 | d008    | 1989-02-10 | 9999-01-01 |
|  10008 | d005    | 1998-03-11 | 2000-07-31 |
|  10009 | d006    | 1985-02-18 | 9999-01-01 |
|  10010 | d006    | 2000-06-26 | 9999-01-01 |
+--------+---------+------------+------------+
10 rows in set (1,068 sec)
```

| Vista | Filas comparadas por gestor | Resultado |
|---|---:|---|
| dept_emp_latest_date | 10 | Coincidencia en las tres columnas |
| current_dept_emp | 10 | Coincidencia en las cuatro columnas |

Las muestras ordenadas coinciden con las salidas de PostgreSQL del apartado
2.3. No se encontraron diferencias en las filas examinadas. Esta comparación
no demuestra por sí sola igualdad del conjunto completo. Los conteos completos
se documentan en el apartado 3.1.

## 3. Consultas de verificación

### 3.1 Comparación de conteos

Las consultas, las salidas de ambos gestores y la comparación por tabla se
documentan en el apartado 1.6. Las seis tablas coinciden en cantidad de filas:
3919015 en cada gestor y ninguna diferencia de conteo. Esta comprobación se
complementará con checksums y pruebas de integridad referencial.

#### Conteos completos de las vistas

Consulta ejecutada en ambos gestores, en MariaDB sobre la base `employees`
y en PostgreSQL conectado a `pdb_employees`. El prefijo `employees` identifica
la base de origen en MariaDB y el esquema de destino en PostgreSQL.

```sql
SELECT 'dept_emp_latest_date' AS vista,
       COUNT(*) AS registros
FROM employees.dept_emp_latest_date
UNION ALL
SELECT 'current_dept_emp',
       COUNT(*)
FROM employees.current_dept_emp
ORDER BY vista;
```

Salida de MariaDB:

```text
+----------------------+-----------+
| vista                | registros |
+----------------------+-----------+
| current_dept_emp     |    300024 |
| dept_emp_latest_date |    300024 |
+----------------------+-----------+
2 rows in set (2,489 sec)
```

Salida de PostgreSQL:

```text
        vista         | registros
----------------------+-----------
 current_dept_emp     |    300024
 dept_emp_latest_date |    300024
(2 filas)
```

| Vista | MariaDB | PostgreSQL | Diferencia |
|---|---:|---:|---:|
| current_dept_emp | 300024 | 300024 | 0 |
| dept_emp_latest_date | 300024 | 300024 | 0 |

Cada vista devuelve 300024 filas en ambos gestores. No existen diferencias
de cantidad en las consultas realizadas. Junto con las muestras del apartado
2.4, estos resultados aportan evidencia de la migración.

### 3.2 Checksums comparables entre motores

#### Tabla departments: comparación completa por fila

Consulta ejecutada en ambos gestores:

```sql
SELECT
    dept_no,
    dept_name,
    MD5(CONCAT(dept_no, '|', dept_name)) AS checksum_fila
FROM employees.departments
ORDER BY dept_no;
```

Se utilizó la misma representación `dept_no|dept_name` y se ordenó por la
clave primaria. Los códigos observados tienen cuatro caracteres y ambas
columnas son obligatorias. Se compararon tanto los valores visibles como
sus huellas MD5 en las nueve filas, sin limitar la consulta a una muestra.

Salida de MariaDB:

```text
+---------+--------------------+----------------------------------+
| dept_no | dept_name          | checksum_fila                    |
+---------+--------------------+----------------------------------+
| d001    | Marketing          | 51f5c8237d5daad581c73972a1fc1db9 |
| d002    | Finance            | ea82f2e117617084a6763b600ed6a849 |
| d003    | Human Resources    | 17052636905bbcd0378b95f69739b153 |
| d004    | Production         | e9e4546494c018703a6986903b9039d3 |
| d005    | Development        | 74d681a5518493e379d147c389b88dcf |
| d006    | Quality Management | 3fdeb57cae9ceb4dc73ac270dabe5bbc |
| d007    | Sales              | 960a64e8a445e529b45a603552b2012d |
| d008    | Research           | 936549cbb095fde4284c9f485b683310 |
| d009    | Customer Service   | 8497d5a13b8bc1f09489af6fdde2a597 |
+---------+--------------------+----------------------------------+
9 rows in set (0,066 sec)
```

Salida de PostgreSQL:

```text
 dept_no |     dept_name      |          checksum_fila
---------+--------------------+----------------------------------
 d001    | Marketing          | 51f5c8237d5daad581c73972a1fc1db9
 d002    | Finance            | ea82f2e117617084a6763b600ed6a849
 d003    | Human Resources    | 17052636905bbcd0378b95f69739b153
 d004    | Production         | e9e4546494c018703a6986903b9039d3
 d005    | Development        | 74d681a5518493e379d147c389b88dcf
 d006    | Quality Management | 3fdeb57cae9ceb4dc73ac270dabe5bbc
 d007    | Sales              | 960a64e8a445e529b45a603552b2012d
 d008    | Research           | 936549cbb095fde4284c9f485b683310
 d009    | Customer Service   | 8497d5a13b8bc1f09489af6fdde2a597
(9 filas)
```

Resultado: las nueve filas coinciden en código, nombre y checksum. No se
detectaron diferencias en la tabla `departments`. MD5 se utiliza como
comprobación de migración, no como garantía criptográfica de identidad.
Este resultado no se extiende a las demás tablas.

#### Tabla dept_manager: comparación completa por fila

Se compararon las 24 filas, ordenadas por la clave primaria compuesta
(emp_no, dept_no). La huella incluye las cuatro columnas de la tabla.
Las fechas se expresaron como YYYY-MM-DD en ambos gestores.

Consulta ejecutada en MariaDB:

```sql
SELECT
    emp_no,
    dept_no,
    MD5(CONCAT(
        emp_no, '|', dept_no, '|',
        DATE_FORMAT(from_date, '%Y-%m-%d'), '|',
        DATE_FORMAT(to_date, '%Y-%m-%d')
    )) AS checksum_fila
FROM employees.dept_manager
ORDER BY emp_no, dept_no;
```

Consulta ejecutada en PostgreSQL:

```sql
SELECT
    emp_no,
    dept_no,
    MD5(CONCAT(
        emp_no, '|', dept_no, '|',
        TO_CHAR(from_date, 'YYYY-MM-DD'), '|',
        TO_CHAR(to_date, 'YYYY-MM-DD')
    )) AS checksum_fila
FROM employees.dept_manager
ORDER BY emp_no, dept_no;
```

Salida de MariaDB (bordes y espacios simplificados, valores conservados):

```text
emp_no | dept_no | checksum_fila
110022 | d001 | aee186baacea419d28ff0455818fed46
110039 | d001 | f2ca4e5165a0064806f1ec7db47b330d
110085 | d002 | ae48171893b42702879bfe30dedce629
110114 | d002 | c9778975805d2872659bf9f362babd3e
110183 | d003 | 47a312c64ea51f1364286c3b17faa0bf
110228 | d003 | e2891c792425ad2ef7dc3f3fc9d55ede
110303 | d004 | 40b1e46ac2d49811fa3b317f913e4baf
110344 | d004 | fc1a9abf292b515f97f50ddcc2399bfa
110386 | d004 | bc10914cfa225881d00a5e4e39241e0d
110420 | d004 | 23456ab6d978c64020b4d9a19a5fad6c
110511 | d005 | e5a1804926bdcd27b67bb5ae43dd8cdf
110567 | d005 | 04a65621830939485b7b2362ce0d1f9a
110725 | d006 | 789b208aaf68ff8e2f0fde127d8ded95
110765 | d006 | 9882208772623a5d0ab9b2fdfd12b233
110800 | d006 | 6bf59c9905691792a50045d4f015bc62
110854 | d006 | 031ad35406dea64880ff6770a7088e3f
111035 | d007 | f8fd71e5cc06910a3114215e86da3560
111133 | d007 | 5c328fbf4540ae52a2dbbb0650eae209
111400 | d008 | 7c78f477711b5ba9323d09a8b85357de
111534 | d008 | 1eaa88259bbf7b80399fdc5b6e9786cc
111692 | d009 | 7c35074b5c6a58a7b7a971a02a9fd1e5
111784 | d009 | b8294cb447875d90efd24d1ef9c049b3
111877 | d009 | 478ea99303c2327913bcd68b8e7d304c
111939 | d009 | ae4678f65c31dfe4f41e62bd3f242746
24 rows in set (0,008 sec)
```

Salida de PostgreSQL (espacios simplificados, valores conservados):

```text
emp_no | dept_no | checksum_fila
110022 | d001 | aee186baacea419d28ff0455818fed46
110039 | d001 | f2ca4e5165a0064806f1ec7db47b330d
110085 | d002 | ae48171893b42702879bfe30dedce629
110114 | d002 | c9778975805d2872659bf9f362babd3e
110183 | d003 | 47a312c64ea51f1364286c3b17faa0bf
110228 | d003 | e2891c792425ad2ef7dc3f3fc9d55ede
110303 | d004 | 40b1e46ac2d49811fa3b317f913e4baf
110344 | d004 | fc1a9abf292b515f97f50ddcc2399bfa
110386 | d004 | bc10914cfa225881d00a5e4e39241e0d
110420 | d004 | 23456ab6d978c64020b4d9a19a5fad6c
110511 | d005 | e5a1804926bdcd27b67bb5ae43dd8cdf
110567 | d005 | 04a65621830939485b7b2362ce0d1f9a
110725 | d006 | 789b208aaf68ff8e2f0fde127d8ded95
110765 | d006 | 9882208772623a5d0ab9b2fdfd12b233
110800 | d006 | 6bf59c9905691792a50045d4f015bc62
110854 | d006 | 031ad35406dea64880ff6770a7088e3f
111035 | d007 | f8fd71e5cc06910a3114215e86da3560
111133 | d007 | 5c328fbf4540ae52a2dbbb0650eae209
111400 | d008 | 7c78f477711b5ba9323d09a8b85357de
111534 | d008 | 1eaa88259bbf7b80399fdc5b6e9786cc
111692 | d009 | 7c35074b5c6a58a7b7a971a02a9fd1e5
111784 | d009 | b8294cb447875d90efd24d1ef9c049b3
111877 | d009 | 478ea99303c2327913bcd68b8e7d304c
111939 | d009 | ae4678f65c31dfe4f41e62bd3f242746
(24 filas)
```

Las 24 claves y sus huellas coinciden entre ambos gestores. No se detectaron
diferencias mediante esta comprobación de todas las filas de dept_manager.
La coincidencia de hashes es evidencia de igualdad del contenido representado,
no una garantía matemática de ausencia de colisiones.

#### Tabla employees: checksum conjunto de todas las filas

Se incluyeron las seis columnas, con fechas en formato YYYY-MM-DD y
longitudes explícitas para delimitar los nombres. Todas las columnas de esta
tabla son NOT NULL. Se calculó MD5 por fila y se concatenaron las huellas,
de 32 caracteres cada una, en orden de emp_no, sin separadores. Finalmente
se calculó MD5 de esa concatenación.

En MariaDB se aumentó el límite de concatenación a 16 MiB solo para la sesión.
La longitud obtenida se comparó con el número de registros multiplicado por
32 para detectar recortes u omisiones de huellas.

Comandos ejecutados en MariaDB:

```sql
SET SESSION group_concat_max_len = 16777216;

WITH filas AS (
    SELECT emp_no,
           MD5(CONCAT(
               emp_no, '|',
               DATE_FORMAT(birth_date, '%Y-%m-%d'), '|',
               CHAR_LENGTH(first_name), ':', first_name, '|',
               CHAR_LENGTH(last_name), ':', last_name, '|',
               gender, '|',
               DATE_FORMAT(hire_date, '%Y-%m-%d')
           )) AS huella
    FROM employees.employees
),
resumen AS (
    SELECT COUNT(*) AS registros,
           GROUP_CONCAT(huella ORDER BY emp_no SEPARATOR '') AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;

SHOW WARNINGS;
```

Salida de MariaDB (formato simplificado):

```text
registros | bytes_calculados | bytes_esperados | checksum_tabla
300024    | 9600768          | 9600768         | b86a17e73f1923dcb2f0a3b20ce69f4f
1 row in set (3,741 sec)
```

Salida de SHOW WARNINGS:

```text
Empty set (0,001 sec)
```

Consulta ejecutada en PostgreSQL:

```sql
WITH filas AS (
    SELECT emp_no,
           MD5(CONCAT(
               emp_no, '|',
               TO_CHAR(birth_date, 'YYYY-MM-DD'), '|',
               CHAR_LENGTH(first_name), ':', first_name, '|',
               CHAR_LENGTH(last_name), ':', last_name, '|',
               gender, '|',
               TO_CHAR(hire_date, 'YYYY-MM-DD')
           )) AS huella
    FROM employees.employees
),
resumen AS (
    SELECT COUNT(*) AS registros,
           STRING_AGG(huella, '' ORDER BY emp_no) AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;
```

Salida de PostgreSQL (formato simplificado):

```text
registros | bytes_calculados | bytes_esperados | checksum_tabla
300024    | 9600768          | 9600768         | b86a17e73f1923dcb2f0a3b20ce69f4f
(1 fila)
```

Los conteos, las longitudes y los checksums coinciden. Se procesaron 300024
filas en cada gestor y la concatenación contiene los 9600768 bytes esperados.
MariaDB no reportó advertencias. No se detectaron diferencias mediante esta
verificación del contenido completo de employees, dentro de las limitaciones
de una comparación basada en hashes MD5.

#### Tabla dept_emp: checksum conjunto de todas las filas

Se incluyeron las cuatro columnas, con fechas en formato YYYY-MM-DD. Las
huellas individuales se concatenaron sin separador en orden de la clave
primaria compuesta (emp_no, dept_no), antes de calcular el checksum conjunto.

Comandos ejecutados en MariaDB:

```sql
SET SESSION group_concat_max_len = 16777216;

WITH filas AS (
    SELECT emp_no, dept_no,
           MD5(CONCAT(
               emp_no, '|', dept_no, '|',
               DATE_FORMAT(from_date, '%Y-%m-%d'), '|',
               DATE_FORMAT(to_date, '%Y-%m-%d')
           )) AS huella
    FROM employees.dept_emp
),
resumen AS (
    SELECT COUNT(*) AS registros,
           GROUP_CONCAT(huella ORDER BY emp_no, dept_no SEPARATOR '') AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;

SHOW WARNINGS;
```

Salida de MariaDB (formato simplificado):

```text
registros | bytes_calculados | bytes_esperados | checksum_tabla
331603    | 10611296         | 10611296        | d8e5cfa497cea51e0e8cd7be31a6f450
1 row in set (3,686 sec)
```

Salida de SHOW WARNINGS:

```text
Empty set (0,000 sec)
```

Consulta ejecutada en PostgreSQL:

```sql
WITH filas AS (
    SELECT emp_no, dept_no,
           MD5(CONCAT(
               emp_no, '|', dept_no, '|',
               TO_CHAR(from_date, 'YYYY-MM-DD'), '|',
               TO_CHAR(to_date, 'YYYY-MM-DD')
           )) AS huella
    FROM employees.dept_emp
),
resumen AS (
    SELECT COUNT(*) AS registros,
           STRING_AGG(huella, '' ORDER BY emp_no, dept_no) AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;
```

Salida de PostgreSQL (formato simplificado):

```text
registros | bytes_calculados | bytes_esperados | checksum_tabla
331603    | 10611296         | 10611296        | d8e5cfa497cea51e0e8cd7be31a6f450
(1 fila)
```

Los 331603 registros, los 10611296 bytes y el checksum coinciden entre ambos
gestores. La longitud calculada es igual a la esperada y MariaDB no reportó
advertencias. No se detectaron diferencias mediante esta comprobación del
contenido completo de dept_emp.

#### Tabla salaries: incidente de truncamiento y verificación por bloques

El primer intento de checksum conjunto devolvió en MariaDB 2844047 registros,
pero solo 16777216 bytes frente a 91009504 esperados. El checksum
fd06543d6e02c8a144d427cd98b6f23b se descartó para comparar la tabla completa.

Advertencia registrada:

```text
Warning | 1260 | Row 524289 was cut by group_concat()
```

Se ejecutó y confirmó el ajuste de sesión:

```sql
SET SESSION group_concat_max_len = 134217728;
SELECT @@SESSION.group_concat_max_len AS limite_bytes;
```

El límite consultado fue 134217728, pero el segundo intento repitió el recorte
a 16777216 bytes. La consulta de diagnóstico fue:

```sql
SELECT VERSION() AS version_mariadb,
       @@SESSION.group_concat_max_len AS limite_concat,
       @@SESSION.max_allowed_packet AS limite_paquete;
```

```text
version_mariadb | limite_concat | limite_paquete
11.8.9-MariaDB  | 134217728     | 16777216
(1 fila; formato simplificado)
```

El límite de paquete coincidía con el tamaño recortado. En PostgreSQL, el
intento conjunto sí había producido los 91009504 bytes esperados y la huella
91cc360c46bc9602f6702d4a16eb73e2, pero no se utilizó para una comparación
con el resultado incompleto de MariaDB. El incidente afectó al cálculo del
checksum, no demostró pérdida de registros ni modificó datos.

Se adoptaron bloques de hasta 100000 filas sin modificar la configuración
global. Cada bloque concatena como máximo 3200000 bytes. La numeración usa
el mismo orden único de la clave primaria (emp_no, from_date) en ambos motores.
Las cuatro columnas se incluyen en las huellas; las fechas usan YYYY-MM-DD.

Consulta ejecutada en MariaDB:

```sql
WITH filas AS (
    SELECT emp_no, from_date,
           FLOOR((ROW_NUMBER() OVER (ORDER BY emp_no, from_date) - 1) / 100000) AS bloque,
           MD5(CONCAT(
               emp_no, '|', salary, '|',
               DATE_FORMAT(from_date, '%Y-%m-%d'), '|',
               DATE_FORMAT(to_date, '%Y-%m-%d')
           )) AS huella
    FROM employees.salaries
),
resumen AS (
    SELECT bloque, COUNT(*) AS registros,
           GROUP_CONCAT(huella ORDER BY emp_no, from_date SEPARATOR '') AS huellas
    FROM filas
    GROUP BY bloque
)
SELECT bloque, registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_bloque
FROM resumen
ORDER BY bloque;

SHOW WARNINGS;
```

Salida de MariaDB (formato simplificado):

```text
bloque | registros | bytes_calculados | bytes_esperados | checksum_bloque
0 | 100000 | 3200000 | 3200000 | c0829f9128db381dd9060a0edf54dd60
1 | 100000 | 3200000 | 3200000 | 0fbc1585ba322a5a67d9a6f3462e1ab1
2 | 100000 | 3200000 | 3200000 | c9079b9309369bed0e453b67c347bba7
3 | 100000 | 3200000 | 3200000 | 6cf7da3d59f3f2e0508153b41d012e7e
4 | 100000 | 3200000 | 3200000 | 21eda07690e146c57bd4bdb9f7c6fda9
5 | 100000 | 3200000 | 3200000 | c6296a4946c9821a08d022de8490735a
6 | 100000 | 3200000 | 3200000 | 43d710e6f8883b6ecfed939bea8ba50d
7 | 100000 | 3200000 | 3200000 | 761db1d0d613ae675871df814e07fcea
8 | 100000 | 3200000 | 3200000 | 77bcbc6d3bfae144fbe4681b585eedaf
9 | 100000 | 3200000 | 3200000 | fcefce9c59b5b5e0e9ae159404ed01e5
10 | 100000 | 3200000 | 3200000 | 04e00bccf6f61aa200ac0aa1c929e4f8
11 | 100000 | 3200000 | 3200000 | bf8db0cfb9d485daadd775f5c9e9beab
12 | 100000 | 3200000 | 3200000 | bab36af28481869f6e59625c74a13a75
13 | 100000 | 3200000 | 3200000 | ab1f27842ce26ca15e7b3791f426b698
14 | 100000 | 3200000 | 3200000 | 62aaa955c743251aeb6f5ccc781976a8
15 | 100000 | 3200000 | 3200000 | 0de5a9ede49df47f4499fa2b95c16682
16 | 100000 | 3200000 | 3200000 | 517940658a5e737b1da6d7ba62f70462
17 | 100000 | 3200000 | 3200000 | 35d05cdba198349af9d57a297218aede
18 | 100000 | 3200000 | 3200000 | d42dc1f3e4990caeb82bbbab3c31db45
19 | 100000 | 3200000 | 3200000 | ec2d9952904266ae6f4b038565af1a52
20 | 100000 | 3200000 | 3200000 | 3a565440bad1ad0891d3e48f5ab61e5f
21 | 100000 | 3200000 | 3200000 | 4ec653c43224e05805121fbdcb558806
22 | 100000 | 3200000 | 3200000 | 8bee030af1ff6bca3966d05d10201cb1
23 | 100000 | 3200000 | 3200000 | 7da8d8449e3861fde02edc43da0a8927
24 | 100000 | 3200000 | 3200000 | f0713179a4a6ac26b4d92fdb9605ebec
25 | 100000 | 3200000 | 3200000 | dcab339f7e42d2bed7d7e199e65bfdaa
26 | 100000 | 3200000 | 3200000 | b08b07358d63fa260082d08663e7e491
27 | 100000 | 3200000 | 3200000 | 280ce584cf1193f4f07c11f8bc7d393c
28 | 44047 | 1409504 | 1409504 | 8b8c337e12dad88ce17fde3e01c3567c
29 rows in set (1 min 10,238 sec)
```

Salida de SHOW WARNINGS:

```text
Empty set (0,021 sec)
```

Consulta ejecutada en PostgreSQL:

```sql
WITH filas AS (
    SELECT emp_no, from_date,
           (ROW_NUMBER() OVER (ORDER BY emp_no, from_date) - 1) / 100000 AS bloque,
           MD5(CONCAT(
               emp_no, '|', salary, '|',
               TO_CHAR(from_date, 'YYYY-MM-DD'), '|',
               TO_CHAR(to_date, 'YYYY-MM-DD')
           )) AS huella
    FROM employees.salaries
),
resumen AS (
    SELECT bloque, COUNT(*) AS registros,
           STRING_AGG(huella, '' ORDER BY emp_no, from_date) AS huellas
    FROM filas
    GROUP BY bloque
)
SELECT bloque, registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_bloque
FROM resumen
ORDER BY bloque;
```

Salida de PostgreSQL (formato simplificado):

```text
bloque | registros | bytes_calculados | bytes_esperados | checksum_bloque
0 | 100000 | 3200000 | 3200000 | c0829f9128db381dd9060a0edf54dd60
1 | 100000 | 3200000 | 3200000 | 0fbc1585ba322a5a67d9a6f3462e1ab1
2 | 100000 | 3200000 | 3200000 | c9079b9309369bed0e453b67c347bba7
3 | 100000 | 3200000 | 3200000 | 6cf7da3d59f3f2e0508153b41d012e7e
4 | 100000 | 3200000 | 3200000 | 21eda07690e146c57bd4bdb9f7c6fda9
5 | 100000 | 3200000 | 3200000 | c6296a4946c9821a08d022de8490735a
6 | 100000 | 3200000 | 3200000 | 43d710e6f8883b6ecfed939bea8ba50d
7 | 100000 | 3200000 | 3200000 | 761db1d0d613ae675871df814e07fcea
8 | 100000 | 3200000 | 3200000 | 77bcbc6d3bfae144fbe4681b585eedaf
9 | 100000 | 3200000 | 3200000 | fcefce9c59b5b5e0e9ae159404ed01e5
10 | 100000 | 3200000 | 3200000 | 04e00bccf6f61aa200ac0aa1c929e4f8
11 | 100000 | 3200000 | 3200000 | bf8db0cfb9d485daadd775f5c9e9beab
12 | 100000 | 3200000 | 3200000 | bab36af28481869f6e59625c74a13a75
13 | 100000 | 3200000 | 3200000 | ab1f27842ce26ca15e7b3791f426b698
14 | 100000 | 3200000 | 3200000 | 62aaa955c743251aeb6f5ccc781976a8
15 | 100000 | 3200000 | 3200000 | 0de5a9ede49df47f4499fa2b95c16682
16 | 100000 | 3200000 | 3200000 | 517940658a5e737b1da6d7ba62f70462
17 | 100000 | 3200000 | 3200000 | 35d05cdba198349af9d57a297218aede
18 | 100000 | 3200000 | 3200000 | d42dc1f3e4990caeb82bbbab3c31db45
19 | 100000 | 3200000 | 3200000 | ec2d9952904266ae6f4b038565af1a52
20 | 100000 | 3200000 | 3200000 | 3a565440bad1ad0891d3e48f5ab61e5f
21 | 100000 | 3200000 | 3200000 | 4ec653c43224e05805121fbdcb558806
22 | 100000 | 3200000 | 3200000 | 8bee030af1ff6bca3966d05d10201cb1
23 | 100000 | 3200000 | 3200000 | 7da8d8449e3861fde02edc43da0a8927
24 | 100000 | 3200000 | 3200000 | f0713179a4a6ac26b4d92fdb9605ebec
25 | 100000 | 3200000 | 3200000 | dcab339f7e42d2bed7d7e199e65bfdaa
26 | 100000 | 3200000 | 3200000 | b08b07358d63fa260082d08663e7e491
27 | 100000 | 3200000 | 3200000 | 280ce584cf1193f4f07c11f8bc7d393c
28 | 44047 | 1409504 | 1409504 | 8b8c337e12dad88ce17fde3e01c3567c
(29 filas)
```

Los 29 bloques coinciden en conteo, longitud y checksum. Los bloques 0 a 27
contienen 100000 filas cada uno y el bloque 28 contiene 44047: suman 2844047
registros y 91009504 bytes de huellas. No hubo recortes en los bloques y
MariaDB no reportó advertencias. No se detectaron diferencias mediante esta
verificación completa por bloques. Sus hashes se comparan bloque a bloque,
no con el hash único del intento anterior.

#### Tabla titles: checksum conjunto y tratamiento de NULL

Se incluyeron las cuatro columnas. Se antepuso la longitud de title para
delimitar el texto y se formatearon las fechas como YYYY-MM-DD. COALESCE
representó to_date nulo con el marcador NULL, distinto de cualquier fecha.
Las huellas se ordenaron por emp_no, title y from_date, usando BINARY en
MariaDB y COLLATE "C" en PostgreSQL para el orden de los títulos.

Comandos ejecutados en MariaDB:

```sql
SET SESSION group_concat_max_len = 16777216;

WITH filas AS (
    SELECT emp_no, title, from_date,
           MD5(CONCAT(
               emp_no, '|',
               CHAR_LENGTH(title), ':', title, '|',
               DATE_FORMAT(from_date, '%Y-%m-%d'), '|',
               COALESCE(
                   DATE_FORMAT(to_date, '%Y-%m-%d'),
                   'NULL'
               )
           )) AS huella
    FROM employees.titles
),
resumen AS (
    SELECT COUNT(*) AS registros,
           GROUP_CONCAT(huella ORDER BY emp_no, BINARY title, from_date SEPARATOR '') AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;

SHOW WARNINGS;
```

Salida registrada de MariaDB (formato simplificado):

```text
Query OK, 0 rows affected (0,011 sec)

registros | bytes_calculados | bytes_esperados | checksum_tabla
443308    | 14185856         | 14185856        | be5f8876ab5d92f1a60bff03ad2fd7ba
1 row in set (5,462 sec)
```

Salida de SHOW WARNINGS:

```text
Empty set (0,001 sec)
```

Consulta ejecutada en PostgreSQL:

```sql
WITH filas AS (
    SELECT emp_no, title, from_date,
           MD5(CONCAT(
               emp_no, '|',
               CHAR_LENGTH(title), ':', title, '|',
               TO_CHAR(from_date, 'YYYY-MM-DD'), '|',
               COALESCE(
                   TO_CHAR(to_date, 'YYYY-MM-DD'),
                   'NULL'
               )
           )) AS huella
    FROM employees.titles
),
resumen AS (
    SELECT COUNT(*) AS registros,
           STRING_AGG(huella, '' ORDER BY emp_no, title COLLATE "C", from_date) AS huellas
    FROM filas
)
SELECT registros,
       OCTET_LENGTH(huellas) AS bytes_calculados,
       registros * 32 AS bytes_esperados,
       MD5(huellas) AS checksum_tabla
FROM resumen;
```

Salida de PostgreSQL (formato simplificado):

```text
registros | bytes_calculados | bytes_esperados | checksum_tabla
443308    | 14185856         | 14185856        | be5f8876ab5d92f1a60bff03ad2fd7ba
(1 fila)
```

Los 443308 registros, los 14185856 bytes y el checksum coinciden. La longitud
calculada es igual a la esperada y MariaDB no reportó advertencias. No se
detectaron diferencias mediante esta comprobación de titles.

#### Resumen de checksums

| Tabla | Filas verificadas por gestor | Método | Resultado |
|---|---:|---|---|
| departments | 9 | Valores y MD5 por fila | Coinciden las 9 filas |
| dept_manager | 24 | MD5 por fila | Coinciden las 24 huellas |
| employees | 300024 | MD5 de huellas ordenadas | Coincide |
| dept_emp | 331603 | MD5 de huellas ordenadas | Coincide |
| salaries | 2844047 | MD5 por bloque de hasta 100000 filas | Coinciden los 29 bloques |
| titles | 443308 | MD5 de huellas ordenadas, con marcador de NULL | Coincide |

Se cubrieron las seis tablas y sus 3919015 registros por gestor. Los resultados
no mostraron diferencias bajo las representaciones utilizadas. Las huellas
MD5 aportan evidencia de consistencia, pero no constituyen una garantía
matemática de identidad ni una prueba de seguridad criptográfica.

### 3.3 Integridad referencial: registros huérfanos

#### Relación dept_emp → employees

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS asignaciones_sin_empleado
FROM employees.dept_emp AS d
LEFT JOIN employees.employees AS e
    ON d.emp_no = e.emp_no
WHERE e.emp_no IS NULL;
```

La consulta cuenta las asignaciones cuyo `emp_no` no tiene correspondencia
en la tabla `employees`.

Salida de MariaDB:

```text
+---------------------------+
| asignaciones_sin_empleado |
+---------------------------+
|                         0 |
+---------------------------+
1 row in set (1,143 sec)
```

Salida de PostgreSQL:

```text
 asignaciones_sin_empleado
---------------------------
                         0
(1 fila)
```

Ambos gestores devolvieron 0: no se encontraron registros huérfanos en esta
relación. Este resultado solo verifica la relación examinada.

#### Relación dept_emp → departments

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS asignaciones_sin_departamento
FROM employees.dept_emp AS d
LEFT JOIN employees.departments AS dep
    ON d.dept_no = dep.dept_no
WHERE dep.dept_no IS NULL;
```

La consulta cuenta las asignaciones cuyo `dept_no` no tiene correspondencia
en `departments`.

Salida de MariaDB:

```text
+-------------------------------+
| asignaciones_sin_departamento |
+-------------------------------+
|                             0 |
+-------------------------------+
1 row in set (0,210 sec)
```

Salida de PostgreSQL:

```text
 asignaciones_sin_departamento
-------------------------------
                             0
(1 fila)
```

Ambos gestores devolvieron 0: no se encontraron asignaciones sin un
departamento correspondiente. Las dos relaciones de `dept_emp` examinadas
no presentan registros huérfanos en las consultas realizadas.

#### Relación dept_manager → employees

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS responsables_sin_empleado
FROM employees.dept_manager AS dm
LEFT JOIN employees.employees AS e
    ON dm.emp_no = e.emp_no
WHERE e.emp_no IS NULL;
```

La consulta cuenta los registros de responsables cuyo `emp_no` no existe
en la tabla `employees`.

Salida de MariaDB:

```text
+---------------------------+
| responsables_sin_empleado |
+---------------------------+
|                         0 |
+---------------------------+
1 row in set (0,002 sec)
```

Salida de PostgreSQL:

```text
 responsables_sin_empleado
---------------------------
                         0
(1 fila)
```

Ambos gestores devolvieron 0: todos los registros de responsables tienen
un empleado correspondiente. No se encontraron huérfanos en esta relación.

#### Relación dept_manager → departments

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS responsables_sin_departamento
FROM employees.dept_manager AS dm
LEFT JOIN employees.departments AS dep
    ON dm.dept_no = dep.dept_no
WHERE dep.dept_no IS NULL;
```

La consulta cuenta los registros de responsables cuyo `dept_no` no existe
en la tabla `departments`.

Salida de MariaDB:

```text
+-------------------------------+
| responsables_sin_departamento |
+-------------------------------+
|                             0 |
+-------------------------------+
1 row in set (0,001 sec)
```

Salida de PostgreSQL:

```text
 responsables_sin_departamento
-------------------------------
                             0
(1 fila)
```

Ambos gestores devolvieron 0: todos los registros de responsables tienen
un departamento correspondiente. Las cuatro relaciones examinadas hasta
este punto no presentan registros huérfanos en las consultas realizadas.

#### Relación salaries → employees

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS salarios_sin_empleado
FROM employees.salaries AS s
LEFT JOIN employees.employees AS e
    ON s.emp_no = e.emp_no
WHERE e.emp_no IS NULL;
```

La consulta cuenta los registros de salarios cuyo `emp_no` no existe en
la tabla `employees`.

Salida de MariaDB:

```text
+-----------------------+
| salarios_sin_empleado |
+-----------------------+
|                     0 |
+-----------------------+
1 row in set (2,657 sec)
```

Salida de PostgreSQL:

```text
 salarios_sin_empleado
-----------------------
                     0
(1 fila)
```

Ambos gestores devolvieron 0: todos los registros de salarios tienen un
empleado correspondiente. No se encontraron huérfanos en esta relación.

#### Relación titles → employees

Consulta ejecutada en ambos gestores:

```sql
SELECT COUNT(*) AS cargos_sin_empleado
FROM employees.titles AS t
LEFT JOIN employees.employees AS e
    ON t.emp_no = e.emp_no
WHERE e.emp_no IS NULL;
```

La consulta cuenta los registros de cargos cuyo `emp_no` no existe en
la tabla `employees`.

Salida de MariaDB:

```text
+---------------------+
| cargos_sin_empleado |
+---------------------+
|                   0 |
+---------------------+
1 row in set (0,860 sec)
```

Salida de PostgreSQL:

```text
 cargos_sin_empleado
---------------------
                   0
(1 fila)
```

Ambos gestores devolvieron 0: no se encontraron cargos sin un empleado
correspondiente.

#### Resumen de integridad referencial

| Relación verificada | Huérfanos en MariaDB | Huérfanos en PostgreSQL |
|---|---:|---:|
| dept_emp → employees | 0 | 0 |
| dept_emp → departments | 0 | 0 |
| dept_manager → employees | 0 | 0 |
| dept_manager → departments | 0 | 0 |
| salaries → employees | 0 | 0 |
| titles → employees | 0 | 0 |

Las seis relaciones de clave foránea examinadas no presentan registros
huérfanos en ninguno de los gestores al momento de las consultas. Estos
resultados confirman la existencia de los registros referenciados, pero no
demuestran por sí solos igualdad de todos los valores entre origen y destino.

### 3.4 Verificación de vistas y conclusiones

Las pruebas iniciales en PostgreSQL se documentan en el apartado 2.3 y las
muestras equivalentes de MariaDB en el apartado 2.4. Las diez filas examinadas
de cada vista coinciden en todas las columnas.
Los conteos completos también coinciden: cada vista devuelve 300024 filas
en ambos gestores (apartado 3.1).
Las seis comprobaciones de registros huérfanos devolvieron 0 en ambos gestores
(apartado 3.3). No se detectaron diferencias de conteo ni referencias sin
correspondencia en las pruebas realizadas.
Los checksums de las seis tablas coinciden según los métodos del apartado
3.2, que incluyen todas las filas y columnas. El recorte encontrado en el
checksum conjunto de salaries se resolvió verificando bloques completos,
sin modificar los datos ni la configuración global del servidor.

En conjunto, los conteos, las huellas y las comprobaciones de huérfanos
aportan evidencia consistente de la migración de tablas. Las dos vistas se
crearon conservando su lógica de consulta y coinciden en conteos y muestras
ordenadas; no se ejecutó una comparación exhaustiva de cada fila de las vistas.
Estas conclusiones corresponden a las consultas realizadas y no equivalen a
una auditoría de permisos, rendimiento o cambios posteriores de los datos.

## 4. Backup de PostgreSQL

### 4.1 Generación del respaldo

Comando ejecutado desde la terminal de Linux:

```bash
pg_dump -h localhost -U juan -W -d pdb_employees -Fc -v -f /home/juan/EntregableFinal/backup_pdb_employees_$(date +%Y-%m-%d_%H-%M).dump
```

La opción `-Fc` genera un archivo de formato personalizado, comprimido por
defecto y restaurable con `pg_restore`. `-W` solicita la contraseña sin
incluirla en el comando; `-v` muestra el progreso. `-f` establece la ruta
de salida y `date` incorpora la fecha y hora al nombre. No se especificaron
filtros de tablas ni opciones de solo estructura o solo datos.

Extracto de la salida recibida:

```text
pg_dump: salvando codificaciones = UTF8
pg_dump: salvando «standard_conforming_strings = on»
pg_dump: salvando «search_path = »
pg_dump: salvando las definiciones de la base de datos
pg_dump: extrayendo el contenido de la tabla «employees.departments»
pg_dump: extrayendo el contenido de la tabla «employees.dept_emp»
pg_dump: extrayendo el contenido de la tabla «employees.dept_manager»
pg_dump: extrayendo el contenido de la tabla «employees.employees»
pg_dump: extrayendo el contenido de la tabla «employees.salaries»
pg_dump: extrayendo el contenido de la tabla «employees.titles»
```

### 4.2 Estado de ejecución y archivo generado

Inmediatamente después del respaldo se consultó el código de salida:

```bash
echo $?
```

```text
0
```

El código 0 indica que `pg_dump` terminó correctamente. Se verificó la
existencia del archivo con:

```bash
ls -lh /home/juan/EntregableFinal/
```

```text
total 35M
-rw-r--r-- 1 juan juan 35M sep 28 23:48 backup_pdb_employees_2026-09-28_23-47.dump
```

El archivo generado es
`/home/juan/EntregableFinal/backup_pdb_employees_2026-09-28_23-47.dump`.
El listado informa un tamaño aproximado de 35M. Esta evidencia confirma
la creación del archivo y la finalización correcta del comando, pero no
sustituye una prueba de restauración.

### 4.3 Inspección del inventario

Comando ejecutado:

```bash
pg_restore --list /home/juan/EntregableFinal/backup_pdb_employees_2026-09-28_23-47.dump
```

Salida registrada:

```text
;
; Archive created at 2026-09-28 23:47:53 -04
;     dbname: pdb_employees
;     TOC Entries: 36
;     Compression: gzip
;     Dump Version: 1.16-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 18.6 (Debian 18.6-1.pgdg13+2)
;     Dumped by pg_dump version: 18.6 (Debian 18.6-1.pgdg12+2)
;
;
; Selected TOC Entries:
;
6; 2615 16496 SCHEMA - employees juan
862; 1247 16498 TYPE employees employees_gender juan
223; 1259 16508 TABLE employees dept_emp juan
228; 1259 24686 VIEW employees dept_emp_latest_date juan
229; 1259 24690 VIEW employees current_dept_emp juan
222; 1259 16503 TABLE employees departments juan
224; 1259 16515 TABLE employees dept_manager juan
225; 1259 16522 TABLE employees employees juan
226; 1259 16531 TABLE employees salaries juan
227; 1259 16538 TABLE employees titles juan
3492; 0 16503 TABLE DATA employees departments juan
3493; 0 16508 TABLE DATA employees dept_emp juan
3494; 0 16515 TABLE DATA employees dept_manager juan
3495; 0 16522 TABLE DATA employees employees juan
3496; 0 16531 TABLE DATA employees salaries juan
3497; 0 16538 TABLE DATA employees titles juan
3324; 2606 16561 CONSTRAINT employees departments idx_16503_primary juan
3327; 2606 16560 CONSTRAINT employees dept_emp idx_16508_primary juan
3330; 2606 16559 CONSTRAINT employees dept_manager idx_16515_primary juan
3332; 2606 16562 CONSTRAINT employees employees idx_16522_primary juan
3334; 2606 16563 CONSTRAINT employees salaries idx_16531_primary juan
3336; 2606 16558 CONSTRAINT employees titles idx_16538_primary juan
3322; 1259 16549 INDEX employees idx_16503_dept_name juan
3325; 1259 16546 INDEX employees idx_16508_dept_no juan
3328; 1259 16545 INDEX employees idx_16515_dept_no juan
3337; 2606 16564 FK CONSTRAINT employees dept_emp dept_emp_ibfk_1 juan
3338; 2606 16569 FK CONSTRAINT employees dept_emp dept_emp_ibfk_2 juan
3339; 2606 16574 FK CONSTRAINT employees dept_manager dept_manager_ibfk_1 juan
3340; 2606 16579 FK CONSTRAINT employees dept_manager dept_manager_ibfk_2 juan
3341; 2606 16584 FK CONSTRAINT employees salaries salaries_ibfk_1 juan
3342; 2606 16589 FK CONSTRAINT employees titles titles_ibfk_1 juan
```

El inventario confirma entradas para el esquema employees, el tipo
employees_gender, seis tablas, seis secciones TABLE DATA, las dos vistas,
seis restricciones de clave primaria, tres entradas adicionales de índices
y seis restricciones de clave foránea. El encabezado identifica el formato
CUSTOM con compresión gzip y versiones 18.6 del servidor y de pg_dump.

La lectura del inventario no restaura objetos ni comprueba por sí sola
todos los datos comprimidos. Por ello se realizó la restauración de prueba
documentada a continuación.

### 4.4 Restauración en una base independiente

Se creó una base nueva sin modificar `pdb_employees`:

```bash
createdb -h localhost -U juan -W -T template0 pdb_employees_prueba
```

El comando solicitó la contraseña y volvió a la terminal sin errores
reportados. Después se ejecutó:

```bash
pg_restore -h localhost -U juan -W \
  -d pdb_employees_prueba \
  --exit-on-error --verbose \
  /home/juan/EntregableFinal/backup_pdb_employees_2026-09-28_23-47.dump
```

La opción `--exit-on-error` detiene la restauración si ocurre un error.
La salida recibida confirmó la creación del esquema, el tipo enumerado,
las seis tablas, las dos vistas, los índices y las restricciones, así como
el procesamiento de los datos de las seis tablas.

Extracto de la salida registrada:

```text
pg_restore: creando SCHEMA «employees»
pg_restore: creando TYPE «employees.employees_gender»
pg_restore: creando TABLE «employees.dept_emp»
pg_restore: creando VIEW «employees.dept_emp_latest_date»
pg_restore: creando VIEW «employees.current_dept_emp»
pg_restore: creando TABLE «employees.departments»
pg_restore: creando TABLE «employees.dept_manager»
pg_restore: creando TABLE «employees.employees»
pg_restore: creando TABLE «employees.salaries»
pg_restore: creando TABLE «employees.titles»
pg_restore: procesando datos de la tabla «employees.departments»
pg_restore: procesando datos de la tabla «employees.dept_emp»
pg_restore: procesando datos de la tabla «employees.dept_manager»
pg_restore: procesando datos de la tabla «employees.employees»
pg_restore: procesando datos de la tabla «employees.salaries»
pg_restore: procesando datos de la tabla «employees.titles»
pg_restore: creando FK CONSTRAINT «employees.dept_emp dept_emp_ibfk_1»
pg_restore: creando FK CONSTRAINT «employees.dept_emp dept_emp_ibfk_2»
pg_restore: creando FK CONSTRAINT «employees.dept_manager dept_manager_ibfk_1»
pg_restore: creando FK CONSTRAINT «employees.dept_manager dept_manager_ibfk_2»
pg_restore: creando FK CONSTRAINT «employees.salaries salaries_ibfk_1»
pg_restore: creando FK CONSTRAINT «employees.titles titles_ibfk_1»
```

Inmediatamente después se verificó el código de salida:

```bash
echo $?
```

```text
0
```

La restauración terminó correctamente en `pdb_employees_prueba`. Esta prueba
confirma que el archivo pudo restaurarse en el entorno utilizado.

### 4.5 Conteos posteriores a la restauración

Conexión a la base de prueba:

```bash
psql -h localhost -U juan -d pdb_employees_prueba
```

Consulta ejecutada:

```sql
SELECT 'departments' AS objeto, COUNT(*) AS registros
FROM employees.departments
UNION ALL
SELECT 'dept_emp', COUNT(*) FROM employees.dept_emp
UNION ALL
SELECT 'dept_manager', COUNT(*) FROM employees.dept_manager
UNION ALL
SELECT 'employees', COUNT(*) FROM employees.employees
UNION ALL
SELECT 'salaries', COUNT(*) FROM employees.salaries
UNION ALL
SELECT 'titles', COUNT(*) FROM employees.titles
UNION ALL
SELECT 'dept_emp_latest_date', COUNT(*)
FROM employees.dept_emp_latest_date
UNION ALL
SELECT 'current_dept_emp', COUNT(*)
FROM employees.current_dept_emp
ORDER BY objeto;
```

Salida registrada:

```text
       objeto        | registros
----------------------+-----------
 current_dept_emp     |    300024
 departments         |         9
 dept_emp            |    331603
 dept_emp_latest_date |    300024
 dept_manager        |        24
 employees           |    300024
 salaries            |   2844047
 titles              |    443308
(8 filas)
```

| Objeto | Antes del respaldo | Después de restaurar | Diferencia |
|---|---:|---:|---:|
| current_dept_emp | 300024 | 300024 | 0 |
| departments | 9 | 9 | 0 |
| dept_emp | 331603 | 331603 | 0 |
| dept_emp_latest_date | 300024 | 300024 | 0 |
| dept_manager | 24 | 24 | 0 |
| employees | 300024 | 300024 | 0 |
| salaries | 2844047 | 2844047 | 0 |
| titles | 443308 | 443308 | 0 |

Los ocho conteos coinciden. Las seis tablas restauradas contienen en total
3919015 registros y ambas vistas pueden consultarse, devolviendo 300024 filas
cada una. Las filas de las vistas no se suman al total de registros de las
tablas, porque son resultados de consultas sobre ellas.

El respaldo se restauró sin errores reportados y conservó los conteos
verificados. Esta prueba no incluye una repetición de los checksums en la
base restaurada; los checksums del apartado 3.2 comparan MariaDB con la base
migrada original, no con `pdb_employees_prueba`.

## Conclusiones

La migración de las seis tablas de MariaDB a PostgreSQL mediante pgloader
se completó sin errores reportados. Los conteos coincidieron en ambos
gestores, con un total de 3919015 registros. También se verificó la
conservación del tipo enumerado `employees_gender` y sus valores `M` y `F`.

Las vistas `dept_emp_latest_date` y `current_dept_emp` se adaptaron mediante
SQL, conservando la lógica de las definiciones originales. Ambas devolvieron
300024 filas en cada gestor y coincidieron en las muestras ordenadas de diez
filas. Estas pruebas respaldan su funcionamiento, aunque no constituyen una
comparación exhaustiva de todos los valores devueltos por las vistas.

Las verificaciones de contenido cubrieron todas las filas y columnas de las
seis tablas mediante checksums comparables, sin detectar diferencias. Además,
las seis consultas de integridad referencial devolvieron cero registros
huérfanos tanto en MariaDB como en PostgreSQL. La coincidencia de hashes se
interpreta como evidencia de consistencia, no como garantía matemática de
identidad absoluta.

Durante la verificación de `salaries` se identificó un truncamiento del texto
concatenado en MariaDB. El problema correspondía al cálculo del checksum y
no demostraba pérdida de datos en la migración. Se resolvió comparando 29
bloques de hasta 100000 filas; todos coincidieron y sus longitudes completas
se comprobaron sin advertencias. Esto evidenció la importancia de revisar
los límites del gestor antes de interpretar diferencias en los resultados.

Finalmente, el respaldo en formato `.dump` se generó y restauró con código
de salida cero en una base independiente. Los conteos de las seis tablas y
las dos vistas restauradas coincidieron con los anteriores al respaldo,
demostrando que el archivo pudo recuperarse en el entorno utilizado. No se
repitieron los checksums sobre la base de prueba, por lo que esta validación
posterior a la restauración se limita a su ejecución correcta y a los conteos.

En conjunto, las evidencias obtenidas respaldan la migración y la recuperación
de la base de datos dentro del alcance de las pruebas realizadas, y permiten
documentar un procedimiento reproducible de transferencia y verificación.

## 5. Entrega en GitHub

Repositorio público de la entrega:

[yugui-mota/migracion-employees-mariadb-postgresql](https://github.com/yugui-mota/migracion-employees-mariadb-postgresql)

### 5.1 Publicación del respaldo

El respaldo se incorporó en la carpeta `backup/` mediante el commit
`0cc002c`, con el mensaje `Agregar backup PostgreSQL verificado`.
Se publicó desde la terminal con:

```bash
git push origin main
```

Extracto de la salida registrada:

```text
Escribiendo objetos: 100% (4/4), 34.74 MiB | 607.00 KiB/s, listo.
To https://github.com/yugui-mota/migracion-employees-mariadb-postgresql.git
   039f3f4..0cc002c  main -> main
```

La actualización de la rama remota confirma la publicación del commit que
contiene el respaldo. El archivo está en
`backup/backup_pdb_employees_2026-09-28_23-47.dump`.





