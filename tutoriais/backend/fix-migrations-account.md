```
python manage.py migrate --fake account 0025_role_description_en_role_description_es_and_more
```

# fix: migrations account 0025_role_description_en_role_description_es_and_more

NOTE: This migration was created automatically by the modeltranslation extension on 2025-02-04 but was not committed.
Because our start-server.sh runs the makemigrations command, the migration file was created directly in the production
pod and applied as 0021. This created divergences between our migration files in the repository and the actual state of
the production database. We fixed this on December 17, 2025 by:
- Deleting the original 0021 migration entry from the django_migrations table
- Creating this migration file with the correct sequence number (0025)
- Running a fake migration to register 0025 in django_migrations without re-executing the SQL


Commit inicial do erro: https://github.com/basedosdados/backend/commit/4916f217c9f67f1a85c85cd882d81c110a0d5d79
Link PR:https://github.com/basedosdados/backend/pull/944