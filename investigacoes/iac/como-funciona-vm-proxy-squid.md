# Como funciona a VM proxy (Squid) na prática
**Issue:** `basedosdados/iac#155`
**Contexto:** [[bloqueio-ip-geografico-datasets]] · [[plano-implementacao-vm-proxy]]

Documentação de referência técnica — como a peça inteira funciona hoje, pra quem for debugar ou estender isso no futuro.

---

## Visão geral do fluxo

```
┌─────────────────────────────── us-central1 (GKE) ───────────────────────────────┐
│                                                                                   │
│   Pod do flow (br_rf_cnpj__dicionario, etc.)                                     │
│                                                                                   │
│   ├── requests.get(url, proxies=brasil_proxy_dict())  ─┐                        │
│   │                                                     │  só as chamadas        │
│   └── httpx.AsyncClient(proxy=brasil_proxy_url())       │  que precisam          │
│                                                          │  de IP brasileiro     │
│   ├── upload pro GCS ────────────────────┐              │                        │
│   ├── query no BigQuery ─────────────────┤  direto,     │                        │
│   ├── heartbeat da API do Prefect ───────┤  sem proxy   │                        │
│   └── leitura do Vault ──────────────────┘              │                        │
│              │                                           │                        │
└──────────────┼───────────────────────────────────────────┼────────────────────────┘
               │                                            │
               │ (egress normal do GKE,                     │ TCP :3128,
               │  IP americano)                             │ Basic Auth
               ▼                                            ▼
        ┌─────────────┐                     ┌──────────────────────────────┐
        │ GCS/BigQuery │                     │   VM brasil-proxy             │
        │ Prefect API  │                     │   southamerica-east1-a        │
        │ Vault        │                     │   IP: <ip-da-vm>                │
        └─────────────┘                     │   Squid (porta 3128)           │
                                              └───────────────┬────────────────┘
                                                               │
                                                               │ CONNECT (HTTPS) /
                                                               │ GET encaminhado (HTTP)
                                                               │ IP brasileiro
                                                               ▼
                                              ┌──────────────────────────────┐
                                              │  arquivos.receitafederal      │
                                              │  .gov.br                      │
                                              └──────────────────────────────┘
```

O ponto central: **só as 3 chamadas de rede que de fato precisam de IP brasileiro passam pela VM** (`data_url()`, `get_table_files()`, `download()` em `br_rf_cnpj/utils.py`, via `brasil_proxy_dict()`/`brasil_proxy_url()`). Todo o resto que o pod faz (upload de dados, consulta ao BigQuery, comunicação com a API do Prefect, leitura de segredos do Vault) sai direto do GKE, sem tocar a VM — é por isso que a variável nunca é setada como `HTTP_PROXY`/`HTTPS_PROXY` global do processo.

## O que é o Squid, e o que ele faz aqui

Squid é um proxy HTTP/HTTPS de propósito geral. Na VM, ele está configurado como um **forward proxy autenticado**, na porta 3128 — não como proxy reverso, não como cache, não com interceptação de TLS (sem "SSL-bump"). A configuração inteira (gerada pelo `metadata_startup_script` da VM, `terraform/brasil_proxy/startup-script.sh.tpl`) é:

```
http_port 3128
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
http_access deny all
```

Traduzindo cada linha:
- `http_port 3128`: onde o Squid escuta.
- `auth_param basic program ...`: usa o helper padrão do Squid (`basic_ncsa_auth`) pra checar usuário/senha contra um arquivo `htpasswd` local (`/etc/squid/passwd`) — o mesmo formato usado por servidores web tradicionais.
- `acl authenticated proxy_auth REQUIRED` + `http_access allow authenticated` + `http_access deny all`: só deixa passar tráfego que já veio autenticado; qualquer requisição sem credencial válida é recusada com `407 Proxy Authentication Required`. Como não há allowlist de IP nesse setup (não existe Cloud NAT gerenciado por Terraform em lugar nenhum, então não dá pra garantir de onde o tráfego do GKE realmente sai), **essa autenticação é o único controle de acesso da VM**.

O usuário/senha em si vêm do `terraform/brasil_proxy/main.tf` (`random_password`, 24 caracteres alfanuméricos — sem caracteres especiais, porque a senha vai embutida numa URL e `[`/`]`/`:`/`@` quebram o parser de URL do Python) e são instalados no arquivo de senha via `htpasswd -cb` no próprio startup script, na primeira vez que a VM sobe.

## Como uma requisição HTTPS atravessa o proxy (sem o Squid nunca ver o conteúdo)

Isso é o ponto mais importante de entender: **o Squid nunca descriptografa nada**. Pra HTTPS (que é o caso da Receita Federal), o fluxo é:

1. O cliente (`requests`/`httpx`, rodando no pod) manda uma requisição `CONNECT arquivos.receitafederal.gov.br:443` pro Squid, autenticada com o usuário/senha.
2. O Squid, depois de validar a autenticação, abre uma conexão TCP crua com o destino (`arquivos.receitafederal.gov.br:443`) e simplesmente **encana bytes** entre as duas pontas — vira um túnel opaco.
3. O handshake TLS (certificado, chaves de sessão, tudo) acontece **direto entre o cliente e o servidor da Receita Federal**, através desse túnel. O Squid só vê bytes criptografados passando, não participa da negociação TLS nem consegue ler o conteúdo.
4. Da perspectiva da Receita Federal, a conexão TLS chegou do IP da VM — é isso que resolve o bloqueio geográfico.

Pra HTTP puro (não é o caso de nenhuma chamada nossa hoje, mas o Squid suporta), o comportamento seria diferente: o Squid recebe a requisição já decodificada e a encaminha ele mesmo, podendo inspecionar/modificar cabeçalhos. Como tudo aqui é HTTPS, isso não se aplica.

Essa característica (nenhuma interceptação TLS) é importante porque `br_bd_execucao_estadual`'s `download_ba.py` tem um workaround de certificado customizado que dependeria de TLS ponta a ponta continuar intacto se algum dia esse dataset também passar a usar o proxy.

## Onde a credencial mora e como ela chega no pod

```
Terraform (random_password)
        │
        ├─► Secret Manager (brasil-proxy-password) ── referência/backup, não é lido em runtime
        │
        └─► montada manualmente em BRASIL_PROXY_URL = http://usuario:senha@<ip-da-vm>:3128
                    │
                    ▼
        kubeseal (encriptado com o cert público do sealed-secrets do cluster)
                    │
                    ▼
        k8s/prefect_workers/{basedosdados,basedosdados-dev}/secret-0X_sealed.yaml
        (chave BRASIL_PROXY_URL dentro do secret gcp-credentials já existente)
                    │
                    ▼  kubectl apply (decriptado só dentro do cluster)
        Secret real "gcp-credentials" no namespace do worker
                    │
                    ▼  envFrom (já configurado no base_job_template do work pool,
                    │   não precisou de nenhuma mudança pra isso)
        Env var BRASIL_PROXY_URL dentro de todo pod novo
                    │
                    ▼
        brasil_proxy_url() / brasil_proxy_dict() (pipelines/utils/utils.py)
                    │
                    ▼
        proxies=.../proxy=... só nas 3 chamadas de rede de br_rf_cnpj/utils.py
```

Pontos que vale destacar aqui:
- A credencial **nunca passa pelo GitHub Actions nem fica salva na configuração de nenhuma deployment do Prefect** — só existe no Secret Manager (registro) e no Secret do Kubernetes (o que realmente é usado).
- `gcp-credentials` é o mesmo secret que já guarda `BASEDOSDADOS_CONFIG`/`BASEDOSDADOS_CREDENTIALS_PROD`/`BASEDOSDADOS_CREDENTIALS_STAGING` (e, em prod, `DBT_SERVICE_ACCOUNT`) — `BRASIL_PROXY_URL` foi só mais uma chave adicionada a ele, seguindo o padrão real de extensão incremental já usado no backend, em vez de criar um secret novo por tema.
- Isso significa que **qualquer flow que rodar a partir de agora já tem a variável disponível**, sem precisar declarar nada em `job_variables` — só o código do dataset decide se usa ou não (via `brasil_proxy_url()`/`brasil_proxy_dict()`).

## O que acontece se a VM cair

Se a VM `brasil-proxy` ficar indisponível (reboot, manutenção, falha), as 3 chamadas de rede de `br_rf_cnpj` que dependem dela simplesmente falham com erro de conexão (timeout ou recusa) — nada mais no sistema é afetado, porque nenhuma outra parte do pod depende do proxy. O Prefect já tem retry configurado nessas tasks (visto nos logs reais: "Retry 1/3... Retry 2/3... Retry 3/3"), então uma instabilidade curta se resolve sozinha; uma queda prolongada da VM exigiria intervenção manual (reiniciar a VM via Terraform/Console, o startup script reconfigura o Squid do zero no boot).
