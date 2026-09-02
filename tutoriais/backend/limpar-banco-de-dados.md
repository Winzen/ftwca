Para facilitar pode ser só excluido todo o nome do container:
```
sudo docker volume rm backend_pgdata
```

Use o 
```
python manage.py dbshell
```
Depois liste todos os databases
```
\l
```

Selecione um diferente do seu atual
```
\c template1
```
Agora você pode excluir o database que estava utilizando:
```
DROP DATABASE IF EXISTS nome_do_banco;
```
```
DROP DATABASE IF EXISTS api_staging;
```
### Caso tenha algum "user" conectado:
Terminei as conexões :
```
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'nome_do_banco' AND pid <> pg_backend_pid();
```
```
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'postgres' AND pid <> pg_backend_pid();
```
Liste as conexões:
```
SELECT pid, datname, usename, application_name, client_addr, backend_start
FROM pg_stat_activity
WHERE datname = 'nome_do_banco';
```
```
SELECT pid, datname, usename, application_name, client_addr, backend_start
FROM pg_stat_activity
WHERE datname = 'postgres';
```
### Crie novamente o DATABASE

```
CREATE DATABASE api_staging;
```
```
\c api_staging
```

### Quando problema do Ricardo acontece na migration

```
ALTER TABLE account_role DROP COLUMN description_en;
ALTER TABLE account_role DROP COLUMN description_es;
ALTER TABLE account_role DROP COLUMN description_pt;
ALTER TABLE account_role DROP COLUMN name_en;
ALTER TABLE account_role DROP COLUMN name_es;
ALTER TABLE account_role DROP COLUMN name_pt;
```
```
ALTER TABLE account_team DROP COLUMN description_en;
ALTER TABLE account_team DROP COLUMN description_es;
ALTER TABLE account_team DROP COLUMN description_pt;
ALTER TABLE account_team DROP COLUMN name_en;
ALTER TABLE account_team DROP COLUMN name_es;
ALTER TABLE account_team DROP COLUMN name_pt;
```