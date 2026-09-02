
### v1.area > v1.entity > v1.entitycategory

v1.area precisa de entity
v1.entity precisa de v1.entitycategory

Sendo assim v1.area, v1.entity e v1.entitycategory podem ser colocados juntos:

```
python manage.py dumpdata v1.area v1.entity v1.entitycategory --indent 2 --output=backup_prod_v1_area_entity.json
```
```
python manage.py loaddata -i backup_prod_v1_area_entity.json
```




Para tentar facilitar:

```
python manage.py dumpdata v1 -e v1.table -e v1.dataset --indent 2 --output=backup_prod_v1_components.json
```
```
python manage.py dumpdata account.account --indent 2 --output=backup_prod_v1_account.json
```
```
python manage.py loaddata -i backup_prod_v1_reforge.json
```


Precisamos pegar uma versão completa do v1, porém fazemos uma alteração para que qualquer contato com a tabela de account seja removido. 

```
python manage.py dumpdata v1 --indent 2 --output=backup_prod_v1_e_contenttypes.json
```

```
python manage.py dumpdata v1 --indent 2 -e account -e contenttypes  --output=backup_prod_v1_e_account.json
```
```
python manage.py loaddata -i backup_prod_v1_e_contenttypes_reforge.json
```
