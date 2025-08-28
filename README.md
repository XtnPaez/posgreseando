# PG cheatsheet propia
Instrucciones y querys para esas cosas que nunca me acuerdo como se hacen en PG

## Esquemas a los que el rol tiene permisos
  ```
  SELECT n.nspname AS esquema,
       r.rolname AS rol,
       has_schema_privilege(r.rolname, n.oid, 'USAGE')  AS puede_usar,
       has_schema_privilege(r.rolname, n.oid, 'CREATE') AS puede_crear
FROM pg_namespace n
CROSS JOIN pg_roles r
WHERE r.rolname IN ('rol_data_analyst','visualizador_sissapp_siempro')
  AND n.nspname NOT LIKE 'pg_%'
  AND n.nspname <> 'information_schema'
ORDER BY rol, esquema;
```

## Tablas a las que el rol tiene permisos
  ```
  SELECT grantee,
       table_schema,
       table_name,
       privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('rol_data_analyst','visualizador_sissapp_siempro')
ORDER BY grantee, table_schema, table_name, privilege_type;
```

## Dar permisos a tablas de un esquema
```
GRANT SELECT ON ALL TABLES IN SCHEMA salud TO rol_data_analyst;
```

## Dar permisos por defecto para las tablas que se creen en el futuro
```
ALTER DEFAULT PRIVILEGES IN SCHEMA salud
GRANT SELECT ON TABLES TO rol_data_analyst;
```

## FDW (Foreign Data Wrapper)

Tenés dos bases:
- bse1 (la que tiene los datos originales).
- bse2 (la que necesita consultar esos datos).

En bse1 hay un esquema salud con la tabla pacientes.

Vos querés que en bse2 aparezca una tabla pacientes (en un esquema, por ejemplo ext_salud) que siempre muestre lo mismo que bse1.salud.pacientes, sin andar duplicando ni sincronizando a mano.

### La lógica paso a paso

Instalar la extensión en la base que consume (bse2)
```
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

Crear un “servidor” que apunte a bse1, esto es como un “alias” de conexión.
```
CREATE SERVER srv_bse1
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'localhost', dbname 'bse1', port '5432');
```

Configurar un mapeo de usuario; Postgres necesita saber con qué usuario conectarse a bse1 cuando alguien en bse2 use el servidor.
```
CREATE USER MAPPING FOR CURRENT_USER
SERVER srv_bse1
OPTIONS (user 'usuario_bse1', password 'secreta123');
```

Importar esquemas o tablas. Podés traer un esquema entero de golpe:
```
IMPORT FOREIGN SCHEMA salud
FROM SERVER srv_bse1
INTO ext_salud;
```

Eso te crea en bse2.ext_salud todas las “foreign tables” que apuntan a las tablas reales en bse1.salud. O, si querés solo una tabla:
```
CREATE FOREIGN TABLE ext_salud.pacientes (
    id     int,
    nombre text,
    edad   int
)
SERVER srv_bse1
OPTIONS (schema_name 'salud', table_name 'pacientes');
```

¿Qué lográs con esto? En bse2, vos hacés:
```
SELECT * FROM ext_salud.pacientes;
```

y en realidad estás leyendo en vivo de bse1.salud.pacientes. Si bse1 cambia (insertás, borrás, actualizás), bse2 lo ve inmediatamente. Es transparente: parece una tabla más en bse2, pero es un “proxy” hacia bse1.

En resumen:
- postgres_fdw = puente nativo de Postgres para leer/escribir entre bases.
- CREATE SERVER = decís dónde está la otra base.
- USER MAPPING = credenciales para conectarse.
- IMPORT FOREIGN SCHEMA = te clona las tablas en un esquema de tu base, y ya las podés usar.

Si tirás un
```
INSERT INTO ext_salud.pacientes (id, nombre, edad)
VALUES (123, 'Cristian', 42);
```

eso viaja por el FDW y se ejecuta en la tabla real: bse1.salud.pacientes.

Sí pega: INSERT, UPDATE, DELETE funcionan en las foreign tables. Pero:
- El rol de bse1 con el que configuraste el USER MAPPING tiene que tener permisos de escritura en la tabla de destino.
- Hay algunas limitaciones (ejemplo: no podés usar TRUNCATE, no podés crear INDEX sobre la foreign table desde bse2, y a veces los RETURNING no funcionan igual).
- Desde el punto de vista del usuario en bse2, parece una tabla local, pero en verdad es un “proxy” hacia bse1.

Entonces:
- SELECT → lee en vivo.
- INSERT/UPDATE/DELETE → se reflejan en bse1.
- DDL (alterar la estructura de la tabla) → no, eso solo en bse1.
