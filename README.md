# Migración de MariaDB a PostgreSQL

Entregable final de **Tecnología de Base de Datos I**.

**Estudiante:** Almendras Davalos Juan Junior  
**Unidad:** Bloque 3 — Migración de un sistema informático a otro SGBD

## Objetivo

Documentar la migración de la base `employees` de MariaDB a PostgreSQL,
incluyendo tablas, datos, vistas, consultas de verificación y un respaldo
restaurable. La base de destino es `pdb_employees` y su esquema es `employees`.

## Archivos de la entrega

| Archivo | Contenido |
|---|---|
| [Informe_Final_Migracion_Employees.md](Informe_Final_Migracion_Employees.md) | Comandos, salidas, adaptación de vistas, verificaciones y conclusiones |
| [backup/backup_pdb_employees_2026-09-28_23-47.dump](backup/backup_pdb_employees_2026-09-28_23-47.dump) | Respaldo PostgreSQL en formato CUSTOM, comprimido con gzip |

## Herramientas utilizadas

- MariaDB 11.8.9.
- PostgreSQL 18.6.
- pgloader 3.6.10~devel para migrar tablas y datos.
- SQL para adaptar y crear las vistas.
- `pg_dump` y `pg_restore` para generar y probar el respaldo.

## Resultados de verificación

| Tabla | Filas en MariaDB | Filas en PostgreSQL |
|---|---:|---:|
| departments | 9 | 9 |
| dept_emp | 331603 | 331603 |
| dept_manager | 24 | 24 |
| employees | 300024 | 300024 |
| salaries | 2844047 | 2844047 |
| titles | 443308 | 443308 |
| **Total** | **3919015** | **3919015** |

- Los checksums comparables coinciden en las seis tablas. Para `salaries`
  se verificaron 29 bloques para evitar el truncamiento de `GROUP_CONCAT`.
- Las seis consultas de registros huérfanos devolvieron 0 en ambos gestores.
- Las vistas `dept_emp_latest_date` y `current_dept_emp` devuelven 300024
  filas cada una y coinciden en las muestras ordenadas de diez filas.
- El backup se restauró en `pdb_employees_prueba` con código de salida 0.
  Los conteos de las seis tablas y las dos vistas restauradas coinciden.

Los hashes aportan evidencia de consistencia, no una garantía matemática
de identidad. No se repitieron los checksums sobre la base restaurada ni
se comparó exhaustivamente cada fila de las vistas.

## Restauración del backup

Se requiere PostgreSQL 18, sus herramientas de cliente y un usuario con
permisos para crear bases y objetos. Ejecutar desde la raíz del repositorio.
Los comandos siguientes usan el usuario `juan`; sustituirlo por el usuario
local de PostgreSQL que corresponda.

Crear una base nueva y vacía:

```bash
createdb -h localhost -U juan -W -T template0 pdb_employees_restaurada
```

Si la base ya existe, detenerse y escoger otro nombre; no sobrescribir una
base existente. Restaurar en la base recién creada:

```bash
pg_restore -h localhost -U juan -W \
  -d pdb_employees_restaurada \
  --no-owner --no-privileges --exit-on-error --verbose \
  backup/backup_pdb_employees_2026-09-28_23-47.dump
```

Estas opciones permiten restaurar con el usuario local sin aplicar los
propietarios ni permisos originales. La prueba documentada en el informe
utilizó el usuario original `juan`, sin estas dos opciones de portabilidad.

Inmediatamente después, consultar el código de salida en la terminal Linux:

```bash
echo $?
```

El valor esperado es 0. Conectarse y comprobar los objetos:

```bash
psql -h localhost -U juan -d pdb_employees_restaurada
```

Dentro de `psql`:

```text
\dt employees.*
\dv employees.*
```

El informe incluye las consultas de conteo para verificar los datos restaurados.
El archivo `.dump` se restaura con `pg_restore`, no con `psql -f`.

## Seguridad

No se incluyen contraseñas en los comandos. Las credenciales de la configuración
de pgloader se sustituyeron por marcadores en el informe. No publicar archivos
de conexión que contengan contraseñas reales.

## Repositorio

[yugui-mota/migracion-employees-mariadb-postgresql](https://github.com/yugui-mota/migracion-employees-mariadb-postgresql)
