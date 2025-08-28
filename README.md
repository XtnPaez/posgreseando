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
