# Plano de implementação — VM proxy em `southamerica-east1`
**Issue:** `basedosdados/iac#155`
**Contexto completo:** [[bloqueio-ip-geografico-datasets]]
**Status:** implementado e validado de ponta a ponta contra a infraestrutura real (VM + secret). Falta só commitar/abrir os PRs (ver seção 4).

---

## 0. Validar o diagnóstico antes de implementar

- [x] Confirmado via logs reais do Prefect 3 (não via teste manual de pod) — `br_rf_cnpj` falha de verdade com `RemoteDisconnected` em runs reais de 2026-09-11 e 2026-09-15, evidência mais forte que qualquer teste isolado teria dado.

## 1. `iac` — Terraform: VM + Squid

- [x] Módulo `terraform/brasil_proxy/` criado (renomeado de `compute_proxy` pra bater com a convenção dos recursos internos).
- [x] `google_service_account` dedicada (`gsa-brasil-proxy`), sem roles de projeto amplos.
- [x] `google_compute_instance` (`e2-small`, `southamerica-east1-a`) com `metadata_startup_script` configurando Squid + Basic Auth.
- [x] `google_compute_address` — IP externo estático provisionado (não publicado aqui; ver output `brasil_proxy_ip` do Terraform ou o Secret Manager).
- [x] `google_compute_firewall` liberando a porta 3128, sem allowlist de IP (autenticação do Squid é o controle principal, confirmado na branch [[bloqueio-ip-geografico-datasets|investigação]]).
- [x] Senha gerada via `random_password`, salva em Secret Manager (`brasil-proxy-password`) e num arquivo local de referência (`brasil-proxy.json`, junto das outras credenciais de service account — nunca commitado).
- [x] `terraform apply -target=module.brasil_proxy` executado com sucesso (precisou de dois imports/retries por um bug conhecido de consistência eventual do provider do Google pra `google_service_account` — resolvido com `terraform import`).
- [x] `curl`/PROPFIND direto contra a VM real confirmaram Squid funcionando e acesso à Receita Federal sem bloqueio (mesmo resultado do teste local com `ngrok`, agora com infra de verdade).

## 2. Distribuir a credencial pro `pipelines` — **mudou completamente do plano original**

O plano original previa uma GitHub Actions secret em `pipelines` + `job_variables["env"]` por flow. Isso foi abandonado depois de revisar o [padrão real de sealed secrets documentado no wiki do `iac`](https://github.com/basedosdados/iac/wiki/Sealed-Secrets) e comparar com o exemplo real do backend (`k8s/website/django/prod/`, onde um único secret `api-prod-secrets` é estendido incrementalmente ao longo de 8 versões, em vez de criar secrets novos por tema).

- [x] Confirmado que o work pool do Prefect 3 já injeta 3 secrets do Kubernetes automaticamente em todo pod, via `envFrom` no `base_job_template` (`gcp-credentials`, `vault-credentials`, `api-keys`) — não documentado em lugar nenhum antes, descoberto consultando a API do Prefect diretamente.
- [x] Decisão: em vez de criar um secret novo dedicado (`brasil-proxy-credentials`, chegou a ser feito e depois desfeito), **estendemos o `gcp-credentials` existente**, adicionando a chave `BRASIL_PROXY_URL` junto das chaves que já estavam lá (`BASEDOSDADOS_CONFIG`, `BASEDOSDADOS_CREDENTIALS_PROD`, `BASEDOSDADOS_CREDENTIALS_STAGING`, e em prod também `DBT_SERVICE_ACCOUNT`) — nenhuma chave antiga foi perdida.
- [x] Gerado `k8s/prefect_workers/basedosdados-dev/secret-03_sealed.yaml` e `k8s/prefect_workers/basedosdados/secret-04_sealed.yaml` (próximo número da sequência de cada namespace), via `kubeseal`, validados com `kubeseal --validate`.
- [x] Aplicados no cluster (`kubectl apply`) — confirmado nos dois namespaces que `BRASIL_PROXY_URL` está no secret real, junto com as chaves antigas intactas.
- [x] ~~PATCH no `base_job_template` do work pool adicionando um 4º `envFrom`~~ — **não é mais necessário**. Como `gcp-credentials` já estava na lista de `envFrom` desde sempre, todo pod novo já recebe `BRASIL_PROXY_URL` automaticamente, sem nenhuma mudança de configuração do work pool.
- [ ] Ainda não commitado no `iac` — arquivos gerados localmente, pendente de commit/push/PR.

## 3. `pipelines` — usar o proxy só onde precisa — **também simplificado**

- [x] `pipelines/datasets/br_rf_cnpj/utils.py`: `_brasil_proxy_url()` lê `BRASIL_PROXY_URL` do ambiente, passado explicitamente como `proxies=`/`proxy=` nas 3 chamadas de rede (`data_url()`, `get_table_files()`, `download()`) — só essa parte do código muda.
- [x] ~~`job_variables["env"]` em `flows.py`~~ — **revertido e removido por completo**. Como a variável agora chega via `envFrom` do secret (seção 2), nenhum flow precisa declarar nada — `flows.py` está de volta idêntico ao `main`.
- [ ] `br_bd_execucao_estadual` (CE/RJ/DF/RO) segue pra depois, sem mudança — mesma decisão de antes.

## 4. Verificação

- [x] Squid + auth funcionando na VM real (`curl` direto).
- [x] PROPFIND contra a Receita Federal via a VM real retornou 207 (sucesso).
- [x] `br_rf_cnpj__dicionario` (a sub-flow mais leve) rodado de ponta a ponta em dev contra a VM real, com `force_run=True`/`materialize_after_dump=False`: baixou os 8 arquivos da Receita Federal via o proxy, processou os relacionamentos entre `dicionario` e as outras 4 tabelas, `Completed`. `flows.py` não precisou de nenhuma mudança — a credencial já chegou sozinha via `envFrom`/`gcp-credentials`.
- [x] Achado e corrigido durante o teste: a senha gerada pelo `random_password` original incluía `[`/`]` no conjunto de caracteres especiais (`override_special = "!#$%&*()-_=+[]{}<>:?"`), e isso quebrava o parser de URL do Python (`ValueError: Invalid IPv6 URL`) quando a senha ia embutida em `http://user:senha@host:porta`. Corrigido trocando pra `special = false` (só alfanumérico, 24 caracteres já dão entropia de sobra) — a VM foi recriada com a senha nova, e o secret no cluster foi atualizado (`merge-and-seal.sh` de novo, preservando as outras chaves).
- [x] Limpeza necessária no meio do processo: a primeira run de teste tinha deixado `job_variables` com a URL do `ngrok` (já morto) salva na configuração da deployment no Prefect — um redeploy sozinho não limpa esse campo (`.deploy(job_variables=None)` não sobrescreve o valor anterior). Precisou de um `PATCH` direto na API do Prefect com `{"job_variables": {"env": []}}` (array vazio, não `{}` — `{}` faz merge e não limpa nada) nas 5 deployments de `br_rf_cnpj` em dev.
- [ ] Falta: commitar as mudanças pendentes em `iac` (o fix da senha em `terraform/brasil_proxy/main.tf`, os dois `secret-0X_sealed.yaml`, `terraform/outputs.tf`) e em `pipelines` (nada pendente além do que já foi commitado — `utils.py` só), e abrir os PRs de verdade nos dois repositórios.
