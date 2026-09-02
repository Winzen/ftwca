Comandos:

```bash
poetry run python manage.py disable_unhealthy_flow_schedules --dry-run
```
```bash
kubectl exec -it api-prod-7fc58b967-swd6v -n website -- poetry run python manage.py disable_unhealthy_flow_schedules
```
### Melhorias:

- [ ] Função pegar lista de ignorados e desativação instantânea de algum form ou arquivo no backend