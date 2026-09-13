# Ideia — gancho de teste personalizado antes da promoção pra prod

Discussão em aberto, 2026-09-13. **Nada implementado ainda** — só o desenho. Pedido do usuário: algumas tabelas precisam de testes personalizados (além do `dbt test` normal), rodando antes da materialização em produção.

## Onde encaixa

Dentro do `mat_test_flow` genérico (`pipelines/utils/metadata/flows.py`), a sequência hoje é:

```
run_dbt(target="dev") → [se falhar, aborta antes de tocar prod]
transfer_files_to_prod_flow(...) → promove pra prod
register_table_materialization_task(...)
```

O gancho entra **depois do `dbt test` de dev passar, antes de `transfer_files_to_prod_flow`** — mesmo ponto onde uma falha de `dbt test` já aborta a run hoje, só que pra validação que não cabe (ou não vale a pena) escrever como teste dbt.

## Por que não dá pra passar a função direto

`mat_test_flow` é **um único deployment genérico, compartilhado por todos os datasets** — não pode ter `if dataset_id == "x"` hardcoded sem quebrar isso. E os parâmetros do disparo (`run_deployment()`) só aceitam dict/tipos nativos (Pydantic valida), nunca um objeto função Python de verdade.

## Desenho proposto: convenção + import dinâmico

Mesmo padrão já usado pra `make_pipeline` (ver [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]]): `mat_test_flow` tenta importar `tasks.py` do dataset e chama uma função por convenção de nome, se ela existir.

```python
# dentro de mat_test_flow, depois do dbt test de dev passar
try:
    tasks_module = importlib.import_module(f"pipelines.datasets.{dataset_id}.tasks")
except ImportError:
    tasks_module = None

custom_test = getattr(tasks_module, "custom_test", None)
if custom_test is not None:
    custom_test(table_id=table_id, env=env)  # levanta exceção -> aborta antes de tocar prod
```

- Dataset sem necessidade especial: não define `custom_test` em `tasks.py`, `getattr` devolve `None`, pula — zero custo, zero mudança pros 11 já migrados.
- Dataset que precisa: só define `custom_test(table_id, env)` no próprio `tasks.py` — mesmo lugar onde `make_check_for_update`/`make_download_data`/`make_pipeline` já moram.
- Levanta exceção → aborta igual um `dbt test` falhando, nunca promove pra prod.

## Diferença de outro gap já documentado — não confundir

Isso é **diferente** do gap do `br_me_cnpj.estabelecimentos` (ver [[migracao-lote-10-datasets]]: o flow antigo também materializava `br_bd_diretorios_brasil.empresa`, outro dataset, **depois** de promover `estabelecimentos` pra prod). Esse aqui:

- **Gancho de antes** (este documento): validação — passa ou levanta exceção, roda antes de prod. Mais simples de generalizar.
- **Gancho de depois** (`br_me_cnpj`): efeito colateral — dispara outro flow/pipeline pra outro dataset, depois de promover. Mais complexo (precisa disparar outro `run_deployment`, não só validar), ainda sem desenho.

## Status

Só desenhado, não implementado. Retomar decidindo: nome da convenção (`custom_test` foi só um chute inicial — pode ser `validate_before_prod`, `extra_validation`, etc.), e se vale implementar já ou esperar até que algum dataset real precise de fato (nenhum dos 11 migrados até agora tem essa necessidade identificada).
