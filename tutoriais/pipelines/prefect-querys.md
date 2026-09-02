
## Ativar flow
```graphql
mutation ($flow_id: UUID!){
  set_schedule_active(input:{flow_id: $flow_id}) {
    success
    error  
    }
  }
```
```json
{
"flow_id": "e2e3c507-e10b-452d-afe6-28f776f833ff"
}
```
## Desativar flow
```graphql
mutation ($flow_id: UUID!){
  set_schedule_inactive(input:{flow_id: $flow_id}) {
    success
    error  
    }
  }
```
```json
{
"flow_id": "e2e3c507-e10b-452d-afe6-28f776f833ff"
}
```
## Query para pegar flow e logs falhos desde de um data.

```graphql
  query($since: timestamptz!) {
    flow(
    where: {
    schedule: { _is_null: false }
    is_schedule_active: {_eq: false}
    flow_runs: {
        state: { _eq: "Failed" }
        start_time: { _gte: $since }
    }
    }
    ) {
    id
    created
    name
    flow_runs(
      where: {
      start_time: { _is_null: false }
      }
    order_by: { start_time: desc }
    limit: 1
    ){
    task_runs(
    where: {
    state: { _in: ["Failed"] }
    task: { 
    name: { _in: ["run_dbt"] }
    }
    
    }
    order_by: { start_time: desc }
    limit: 1 ) {
    id
    state
    end_time
    state_message
    logs(
          where: {
            message: {
              _ilike: "%Custom quota exceeded%"
            }
          }
        ){
      message
    }
    task 
      {
    id
    name
    }
    }
    }
    }
    }      
```
```json
{
  "since": "2026-02-11T08:03:26.213827+00:00"
}
```
## Mudar valores em um flow

```graphql
mutation UpdateFlowCreated($flow_id: uuid!, $created: timestamptz){
  update_flow_by_pk(
    pk_columns: { id: $flow_id }
    _set: { created: $created }
  ) {
    id
    name
    created
  }
}
```
```json
{
"flow_id": "e2e3c507-e10b-452d-afe6-28f776f833ff",
"created": "2026-02-12T10:00:00+00:00"
}
```
- `id`
- `flow` (FK)
- flow_link
- `triggered_by`
- `reason`
- `created_at`