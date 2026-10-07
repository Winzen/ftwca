# Teste local do proxy com ngrok (antes de provisionar a VM)
**Issue:** `basedosdados/iac#155`
**Contexto:** [[bloqueio-ip-geografico-datasets]] · [[plano-implementacao-vm-proxy]]
**Status:** executado com sucesso em 2026-09-22 — ver [[#Resultado real (2026-09-22)]]

---

## Objetivo

Validar o desenho inteiro (Squid + `job_variables["env"]` + `proxies=` explícito no código) usando o computador de quem está testando como proxy temporário, **sem provisionar nada em GCP ainda**. Se funcionar, dá confiança pra investir na VM real; se não, descobrimos o problema (de config, de código, ou até que o bloqueio não é geográfico de verdade) sem ter gastado nada.

Pré-requisito: confirmar antes que o bloqueio é geográfico de verdade (passo 0 do [[plano-implementacao-vm-proxy]] — um `curl -X PROPFIND` contra `arquivos.receitafederal.gov.br` de uma máquina no Brasil, comparado com o mesmo comando de dentro do cluster, que já sabemos que falha).

## Passo a passo

### 1. Subir o Squid localmente (Docker)

```bash
mkdir -p /tmp/squid-test
cat > /tmp/squid-test/squid.conf << 'EOF'
http_port 3128
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
http_access deny all
EOF

docker run -d --name squid-test \
  -p 3128:3128 \
  -v /tmp/squid-test/squid.conf:/etc/squid/squid.conf \
  ubuntu/squid

# criar usuário/senha de teste dentro do container
docker exec squid-test htpasswd -cb /etc/squid/passwd testuser "$(openssl rand -hex 12)"
docker exec squid-test cat /etc/squid/passwd   # confirma o hash, guarde a senha em texto plano que você gerou acima
docker restart squid-test
```

Teste local (antes de expor): `curl -x http://testuser:<senha>@localhost:3128 https://ifconfig.me` deve retornar seu próprio IP.

### 2. Expor via ngrok

```bash
ngrok tcp 3128
```

O ngrok imprime um endereço público temporário, tipo `tcp://0.tcp.ngrok.io:12345`. Anote host e porta — isso substitui `<ip-da-vm>:3128` do desenho final.

**Nota de segurança**: enquanto o túnel estiver de pé, a porta fica exposta publicamente na internet — mitigado pela autenticação do Squid (usuário/senha aleatórios, só pra esse teste). Derrubar o túnel e o container assim que o teste terminar (passo 5).

### 3. Confirmar o proxy através do túnel

```bash
curl -x http://testuser:<senha>@0.tcp.ngrok.io:12345 https://ifconfig.me
```
Deve retornar o mesmo IP do passo 1 (o seu, presumivelmente brasileiro) — agora acessível de fora da sua rede local.

### 4. Rodar o flow de `br_rf_cnpj` em `dev` apontando pro túnel

- Fazer a mudança de código descrita no [[plano-implementacao-vm-proxy]] (seção 3) em `pipelines/datasets/br_rf_cnpj/utils.py` — ler `BRASIL_PROXY_URL` do ambiente e passar como `proxies=`/`proxy=` nas 3 chamadas de rede.
- Numa branch de teste, setar temporariamente em `pipelines/datasets/br_rf_cnpj/flows.py` (só numa das 5 sub-flows, ex. `estabelecimentos`, pra um teste rápido):
  ```python
  br_rf_cnpj__estabelecimentos.job_variables = {
      "env": [
          {"name": "BRASIL_PROXY_URL", "value": "http://testuser:<senha>@0.tcp.ngrok.io:12345"},
      ],
  }
  ```
  (valor hardcoded só pra esse teste manual — no desenho final isso vem de uma GitHub Actions secret, nunca commitado; aqui não faz commit desse valor, é só pra rodar localmente/testar o deploy em dev.)
- Deploy desse flow em `dev` (mesmo processo já usado pra testar outros flows migrados, ver `.claude/skills/pipeline-doctor/` do repo `pipelines`) e disparar uma run manual.

### 5. Verificar e derrubar

- Conferir nos logs do Prefect que a chamada `data_url()`/PROPFIND não quebra mais com `RemoteDisconnected`.
- Derrubar tudo assim que confirmar: `docker rm -f squid-test`, encerrar o `ngrok`, reverter o `job_variables` de teste (não deixar o valor hardcoded do proxy commitado em lugar nenhum).

## Resultado esperado

- **Sucesso** → confiança de que o desenho funciona de ponta a ponta; segue pro [[plano-implementacao-vm-proxy]] pra provisionar a VM real em `southamerica-east1`.
- **Falha com o mesmo erro de antes** → ou o Squid/túnel não está configurado certo (revisar passo 3), ou o bloqueio da Receita Federal não é só por IP (ex. depende de algum outro sinal, cookie, TLS fingerprint) — vale investigar antes de gastar em VM.

## Resultado real (2026-09-22)

Executado de ponta a ponta, com sucesso completo. Diferenças reais em relação ao passo a passo acima (documentadas aqui pra quem for repetir o teste depois):

### Ambiente
- WSL2 + Docker Desktop (via `docker.exe`, não o `docker` nativo do WSL — o daemon do WSL não estava integrado). Mount de volume só funcionou com caminho no formato Windows (`D:\...`), não `/mnt/d/...` — o Docker Desktop não resolveu o path WSL diretamente.
- A imagem `ubuntu/squid` não tem `htpasswd` instalado. Gerado o hash com `openssl passwd -1 -salt <salt> <senha>` (MD5-crypt, `$1$...`) direto no host, escrito num arquivo `passwd` e montado no container — sem precisar instalar nada a mais.
- ngrok exigiu cadastro de cartão de crédito/débito pra liberar endpoints TCP no plano grátis (não cobra, só verificação antiabuso — `ERR_NGROK_8013`). Sem isso, `ngrok tcp` falha mesmo com authtoken válido.
- O túnel caiu num edge `0.tcp.sa.ngrok.io` (América do Sul) — bônus de latência, não planejado.

### Mudança de desenho: não hardcoded, lido do ambiente em tempo de deploy
Em vez do `job_variables` hardcoded do passo 4 (só pra teste manual), a implementação real (branch `test/proxy-br-rf-cnpj`, PR ainda não aberto) já ficou no formato final desde o começo: `pipelines/datasets/br_rf_cnpj/flows.py`'s `_rf_cnpj_flow()` lê `os.environ.get("BRASIL_PROXY_URL", "")` no momento do deploy e embute isso no `job_variables["env"]` de todas as 5 sub-flows (a fábrica é compartilhada, não só uma). O valor real (URL do ngrok + credenciais do Squid) foi passado só como variável de ambiente do shell que rodou `deploy_flows.py` localmente — nunca commitado em lugar nenhum.

Deploy feito com o script real do repo, não um passo manual:
```bash
BRASIL_PROXY_URL="http://testuser:<senha>@0.tcp.sa.ngrok.io:27745" \
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch test/proxy-br-rf-cnpj \
  --files pipelines/datasets/br_rf_cnpj/flows.py
```
Registrou as 5 deployments (`dev-br_rf_cnpj__*`) sem erro.

### Teste 1 — só o check (`br_rf_cnpj__estabelecimentos`)
Run disparada via `run_deployment` (Prefect). Completou (`COMPLETED`) e o log mostrou a chamada que antes quebrava com `RemoteDisconnected` (`data_url()`, `utils.py:41`) funcionando normalmente:
```
A data máxima extraida da API da Receita Federal que será utilizada para comparar com os metadados da BD: 2026-09
```
Como a data da fonte já batia com a coverage existente, o flow concluiu sem baixar nada — validou o proxy, mas não o download.

### Teste 2 — ponta a ponta de verdade (`br_rf_cnpj__dicionario`)
Escolhido por ser a sub-flow mais leve: `dicionario` processa 8 arquivos de referência pequenos (`qualificacoes`, `paises`, `motivos`, etc.), não os registros grandes (`empresas`/`estabelecimentos`/`socios`/`simples`, potencialmente GBs — inadequados pra rodar pela internet residencial + túnel grátis).

Disparado com `force_run=True` (pula o check de "já está atualizado") e `materialize_after_dump=False` (só dump + dbt em `dev`, sem tocar produção — o objetivo era testar o proxy, não materializar dados de verdade). Resultado, ~5 minutos depois, `COMPLETED`, com toda a cadeia real exercitada:
1. Baixou os 8 arquivos da Receita Federal via proxy (`Qualificacoes.zip`, `Paises.zip`, `Motivos.zip`, etc.) — todos com sucesso.
2. Extraiu, carregou e processou os relacionamentos entre `dicionario` e as outras 4 tabelas.
3. Upload pro bucket de dev (`gs://basedosdados-dev/staging/br_rf_cnpj/dicionario`) — confirmando que só a chamada à Receita Federal passa pelo proxy, esse upload não passou (como pretendido no desenho).
4. `dbt run` e `dbt test` contra `target=dev`, os dois OK.

### Conclusão
O desenho inteiro funciona de ponta a ponta, incluindo download real de arquivos através do proxy, não só a checagem de data. Dá confiança suficiente pra seguir pro [[plano-implementacao-vm-proxy]] e provisionar a VM real — a VM só precisa reproduzir o que o Squid local já provou que funciona (forward proxy simples, auth por usuário/senha, sem SSL-bump).

Squid e túnel ngrok derrubados ao final do teste (`docker rm -f squid-test`, processo `ngrok` encerrado).
