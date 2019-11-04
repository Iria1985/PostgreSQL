# PostgreSQL
## VARIABLES USADAS 
appli_url: "nombredelapli"
db_postgres_compose_name: "{{appli_url}}-db"
db_image_save: "postgres:11-alpine"
db_postgres_database_name: "nombrebasedatos"
db_postgres_user_name: "usuarioadmin"
db_postgres_user_password: "passusuarioadmin"
db_postgres_user_backup: "usuariobackup"
db_postgres_password_backup: "passusuariobackup"

## DOS MANERAS DE HACER BACKUP:

### PG_BASEBACKUP: Backup Completo.

#### SCRIPT DE CREACIÓN DE USUARIO CON PERMISOS PARA BASE BACKUP
```
#!/bin/bash
export PGPASSWORD="{{db_postgres_user_password}}"
psql -h {{db_postgres_compose_name}} -U {{ db_postgres_user_name }} -d {{ db_postgres_database_name }} -c "CREATE USER {{ db_postgres_user_backup }} WITH SUPERUSER NOINHERIT NOCREATEROLE NOCREATEDB LOGIN REPLICATION PASSWORD '{{ db_postgres_password_backup }}'"
echo -ne "host\treplication\tall\t\t 0.0.0.0/0\t\tmd5\n" >> "$PGDATA/pg_hba.conf"
echo -ne "port = 5432" >> "$PGDATA/postgres.conf"
```
#### COMANDO PG_BASEBACKUP
echo "host replication all 0.0.0.0/0 md5" >> "$PGDATA/pg_hba.conf" && export PGPASSWORD="{{db_postgres_password_backup}}" && pg_basebackup -U {{ db_postgres_user_backup }} -h {{ db_postgres_compose_name }} -p {{ db_postgres_port }} --format=tar --gzip --compress=9 --progress --pgdata /db_backups/


### PG_DUMP: Backup básico de datos.

#### SCRIPT DE CREACIÓN DE USUARIO CON PERMISOS PARA BASE BACKUP
```
#!/bin/bash
export PGPASSWORD="{{db_postgres_user_password}}"
psql -h {{db_postgres_compose_name}} -U {{ db_postgres_user_name }} -d {{ db_postgres_database_name }} -c "CREATE USER {{ db_postgres_user_backup }} WITH SUPERUSER NOINHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION PASSWORD '{{ db_postgres_password_backup }}'
```
#### COMANDO PG_DUMP
```
export PGPASSWORD="{{db_postgres_user_password}}" && pg_dump -U {{ db_postgres_user_name }} -h {{ db_postgres_compose_name }} -p {{ db_postgres_port }} -d {{ db_postgres_database_name }} > /db_backups/cronned_backup_postgres_db.sql  
```
### PG_DUMPALL: Todas las bases de datos y el contenido + tablespaces

#### SCRIPT DE CREACIÓN DE USUARIO CON PERMISOS PARA BASE BACKUP
```
#!/bin/bash
export PGPASSWORD="{{db_postgres_user_password}}"
psql -h {{db_postgres_compose_name}} -U {{ db_postgres_user_name }} -d {{ db_postgres_database_name }} -c "CREATE USER {{ db_postgres_user_backup }} WITH SUPERUSER NOINHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION PASSWORD '{{ db_postgres_password_backup }}'
```
#### COMANDO PG_DUMP
```
export PGPASSWORD="{{db_postgres_user_password}}" && pg_dumpall -U {{ db_postgres_user_name }} -h {{ db_postgres_compose_name }} -p {{ db_postgres_port }}  > /db_backups/cronned_backup_postgres_db.sql  
```
#### RESTAURACIÓN DE LAS BASES DE DATOS:

1. Restaurando PG_BASEBACKUP
```
#!/bin/bash
#RESTORE PG_BASEBACKUP 
echo "Give me the name of your pg_wall"
read -p wall
echo "Give me the name of your base"
read -p base
docker stop {{db_postgres_container_name}}
docker rm -f {{db_postgres_container_name}}
docker run -it --name {{db_backup_aux_name}} -v {{db_postgres_vol_name}}:/var/lib/postgresql/data/ -v {{appli_update_path}}:/save alpine sh
rm -rf  /var/lib/postgresql/data
exit
docker rm -f {{db_backup_aux_name}}
cd {{appli_remote_root_path}}
docker-compose up -d
docker stop {{db_postgres_container_name}}
docker run -it --name {{db_backup_aux_name}} -v {{db_postgres_vol_name}}:/var/lib/postgresql/data/ -v {{appli_update_path}}:/save alpine sh
tar xzvpf /save/$wal -C /var/lib/postgresql/data/pg_wal
tar xzvpf /save/$base -C /var/lib/postgresql/data/
exit
docker rm -f {{db_backup_aux_name}}
docker start {{db_postgres_container_name}}
```
2. Restaurando PG_DUMP
```
cat /db_restore/{{backup_nasme}} |psql -U {{ db_postgres_user_backup }} -h {{ db_postgres_compose_name }} -p {{ db_postgres_port }} -d {{ db_postgres_database_name }}
```
3. Restaurando PG_DUMPALL
```
psql -U {{ db_postgres_user_backup }} -h {{ db_postgres_compose_name }} -p {{ db_postgres_port }} -d {{ db_postgres_database_name }} -f /db_restore/{{backup_name}}
```
