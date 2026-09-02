```
uv run dbt run --select models/br_bd_diretorios_brasil/br_bd_diretorios_brasil__empresa.sql
```

```
kubectl get pod prefect-job-1c8fceab-94r2n -n prefect-agent-basedosdados -o jsonpath='{.spec.serviceAccountName}'
```
```
 kubectl apply -f debug_pod_flow_prod.yaml
```
```
 kubectl get pods -n prefect-agent-basedosdados
```
```
kubectl exec -it debug-prefect-flow -n prefect-agent-basedosdados -- bash
```
```
kubectl cp prefect-agent-basedosdados/debug-prefect-flow:/app/target ./target
```
```
kubectl delete pod debug-prefect-flow -n prefect-agent-basedosdados
```


```
kubectl exec -it prefect-job-9fe475d5-5h8qq -n prefect-agent-basedosdados -- printenv
```