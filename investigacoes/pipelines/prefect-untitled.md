```
query($names: [String!]) {
  flow(
    where: {
      name: { _in: $names }
      is_schedule_active: {_eq: true}
    }
    order_by: { version: desc }
  ) {
    id
    name
    is_schedule_active
    archived
    version
    created
  }
}
```
```
{
  "names": "br_rj_isp_estatisticas_seguranca.armas_apreendidas_mensal"
}
```

  query($flow_id: uuid!) {
    flow_group(where: {id: { _eq: $flow_id }}){
		flows{
      id
      name
      is_schedule_active
      schedule 
    }
    }
    
    }
{
  "flow_id": "88da00c0-17d3-49ca-a78d-17d6df9bda48"
}

 query($name: String) {
		flow(where: {name: { _eq: $name }}){
      id
      name
      is_schedule_active
      schedule 
    }
    }
    
{
  "name": "br_rj_isp_estatisticas_seguranca.armas_apreendidas_mensal"
}