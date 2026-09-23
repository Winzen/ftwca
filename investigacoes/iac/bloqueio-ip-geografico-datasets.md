# Bloqueio de IP geográfico em fontes de dados — VM proxy em `southamerica-east1`
**Data:** 2026-09-22
**Issue:** `basedosdados/iac#155`
**Status:** implementado, validado de ponta a ponta e com PRs abertos (`iac#156`, `pipelines#2094`) — ver [[plano-implementacao-vm-proxy]] pro detalhamento da implementação e [[como-funciona-vm-proxy-squid]] pra referência técnica de como o Squid e a distribuição de credencial funcionam na prática

---

## Contexto

O cluster GKE de produção roda em `us-central1`, então toda requisição de saída de um flow sai com IP americano. Algumas fontes de dados bloqueiam esse tráfego por geolocalização, o que quebra flows que dependem delas.

A issue `iac#155` já levantou o problema e comparou duas soluções, com custo estimado:

### Opção 1 — Cluster GKE novo em `southamerica-east1` (São Paulo)
GKE não permite um cluster com node pools espalhados em regiões diferentes, só zonas dentro da mesma região — então ter compute de verdade em São Paulo exige um cluster novo, não só mais um node pool.

**Custo estimado**: ~$95-115/mês (taxa fixa de gerenciamento do cluster ≈ $73/mês domina o custo, existe independente de uso).

### Opção 2 — VM proxy em `southamerica-east1` (recomendada)
Mantém o compute normal em `us-central1-c`; só o tráfego HTTP da(s) fonte(s) bloqueada(s) passa por um proxy numa VM pequena em São Paulo.

**Custo estimado**: ~$10-25/mês.

### Decisão
Optamos pela VM proxy (opção 2): resolve o problema real (fontes específicas bloqueadas, não todo o tráfego), custa 4-8x menos, e não cria uma segunda peça de infraestrutura pesada (cluster GKE) pra manter. A opção de migrar *toda* a infraestrutura pra São Paulo também foi considerada e descartada — o custo de compute sobe ~1.5-1.6x permanentemente pra tudo, e significaria refazer boa parte do que acabou de ser concluído na migração Prefect 0→3.

---

## O caso real mais urgente: `br_rf_cnpj`

A issue original foi escrita em cima do caso de `br_bd_execucao_estadual` (estados CE, RJ, DF bloqueados, documentado em `pipelines/models/br_bd_execucao_estadual/ONBOARDING_PLAN.md`), hoje contornado com mirror congelado (seed único, sem flow rodando de verdade em produção continuamente) — sem urgência real.

Só depois, ao revisitar o problema, identificamos que **`br_rf_cnpj` (Cadastro Nacional de Pessoa Jurídica) está ativamente quebrado em produção** pelo mesmo motivo. Confirmado via logs reais do Prefect 3 (`get_failed_flow_runs`): as 5 sub-flows (`estabelecimentos`, `dicionario`, `simples`, `socios`, `empresas`) falham repetidamente com

```
ConnectionError: ('Connection aborted.', RemoteDisconnected('Remote end closed connection without response'))
```

ao chamar `arquivos.receitafederal.gov.br`. Falhas recorrentes em 2026-09-11 e 2026-09-15, sempre no mesmo ponto de código:
- `pipelines/datasets/br_rf_cnpj/utils.py:41` — `data_url()`, requisição `PROPFIND` (WebDAV)
- `pipelines/datasets/br_rf_cnpj/utils.py:100` — `get_table_files()`, mesma requisição `PROPFIND`
- `pipelines/datasets/br_rf_cnpj/utils.py:224` — `download()`, `httpx.AsyncClient()` baixando os ZIPs de verdade

Diferente de `br_bd_execucao_estadual`, esse é um flow de produção contínuo, não um seed único — é o caso que justifica priorizar a implementação da VM proxy.

## Nem todo "bloqueio" é geográfico de verdade

Vale registrar um precedente do próprio `br_bd_execucao_estadual`: o estado do Rio Grande do Sul (RS) estava na mesma lista de "bloqueados" dessa fonte, mas o histórico do dataset (`pipelines/models/br_bd_execucao_estadual/code/constants.py`, docstring de `flows.py`) mostra que o problema real era o *caminho de rede* de um teste anterior — a fonte recusava um IP residencial australiano mas respondia normalmente de um range universitário, não era um bloqueio de país. Testando direto do GKE, RS funcionou sem proxy nenhum e foi liberado em produção em 2026-09-10.

Isso não invalida o caso do `br_rf_cnpj` (o padrão `RemoteDisconnected` consistente e repetido em datas diferentes é uma evidência bem mais forte de bloqueio real do que o que existia pro RS), mas reforça que vale testar a reachability direto do cluster antes de assumir que qualquer bloqueio é geográfico — ver primeiro passo do plano de implementação.

## Achado técnico que simplificou o desenho

O work pool Kubernetes do Prefect 3 já suporta injetar variáveis de ambiente por flow, via `job_variables["env"]` (lista de `{name, value}`, não dict plano — um dict já quebrou validação de schema no passado, `pipelines#1893`). Existe precedente real e recente em `pipelines/datasets/br_sfb_sicar/flows.py:501-508` usando isso pra `MALLOC_ARENA_MAX`. Isso elimina a necessidade de qualquer wrapper central de proxy em `pipelines/utils/` — o mecanismo de configuração por flow já existe, só falta usá-lo pro proxy.

Detalhes completos da implementação em [[plano-implementacao-vm-proxy]]. Validação local antes de provisionar a VM em [[teste-local-proxy-ngrok]].

## Tailscale — o que é e por que apareceu nessa discussão

Ao planejar como testar o proxy localmente antes de provisionar a VM real, avaliamos o Tailscale como alternativa ao `ngrok` pra esse teste.

**O que é**: uma VPN mesh baseada em WireGuard. Cada dispositivo (um computador, um pod de um cluster, um servidor) roda um cliente Tailscale e entra numa rede privada virtual comum. Depois disso, os dispositivos se enxergam diretamente uns aos outros por um IP privado fixo (ex. `100.x.y.z`), criptografado, **mesmo atrás de NAT/roteador residencial**, sem precisar abrir porta no roteador nem ter IP público — o NAT traversal é resolvido pela própria ferramenta. É o oposto do modelo de VPN tradicional (servidor central, IPs de túnel, firewall manual): basta autenticar o cliente em cada máquina.

**Estado atual no `iac`**: existe um chart Helm em `k8s/tailscale/chart/`, com `decrypt.sh`/`encrypt.sh` e um `values.yaml.gpg` criptografado. Isso confirma que o Tailscale já é usado pra alguma coisa dentro do cluster, mas **não conseguimos confirmar pra quê** — os valores estão criptografados, e não há nenhuma referência a Tailscale nos manifests de `k8s/prefect_workers/` (nem `basedosdados/` nem `basedosdados-dev/`), então não há evidência de que os pods de worker do Prefect já estejam nessa malha. Pode ser usado só pra acesso administrativo ao cluster, por exemplo.

**Por que não usamos agora**: como não dá pra confirmar se os workers já estão na malha Tailscale sem acesso à chave de decriptação, optamos por validar o teste local com `ngrok` primeiro (mais rápido de configurar, não depende de infraestrutura já existente). Tailscale continua sendo a opção "mais correta" pro longo prazo (evita expor o Squid publicamente mesmo que temporariamente), vale revisitar se/quando confirmarmos que os workers já estão nessa rede.

## Evolução futura considerada: work pool dedicado no Brasil (não implementado)

Durante a discussão do desenho, surgiu uma pergunta de cenário ideal: em vez de cada dataset precisar configurar explicitamente `proxies=`/`BRASIL_PROXY_URL` no código pra contornar o bloqueio, dava pra ter uma **tag** (ex. `brasil`) que fizesse o pod do flow já subir com IP brasileiro nativamente, sem nenhuma configuração de proxy por dataset?

**Resposta**: sim, mas isso exige compute real rodando fisicamente no Brasil — não dá pra fazer só com uma VM proxy, porque nesse desenho o pod continua em `us-central1`; só a chamada de rede explicitamente configurada é redesviada. Pra qualquer requisição HTTP do pod sair nativamente com IP brasileiro, o processo do flow em si precisa estar rodando numa máquina que já tem essa saída de rede.

### O que é um work pool (Prefect 3)

Um work pool é a fila que agrupa "onde e como" um flow roda. Cada deployment é atribuído a um work pool; um **worker** (processo que fica de olho na fila) pega as runs agendadas e executa na infraestrutura configurada pra aquele pool. Hoje o projeto já usa esse mecanismo: dois work pools Kubernetes, `basedosdados` (prod) e `basedosdados-dev` (dev), cada um com seu próprio `KubernetesWorker` rodando dentro do GKE em `us-central1`, criando um Job K8s no namespace correspondente a cada run.

### A ideia

Um terceiro work pool (ex. `basedosdados-brasil`), com o worker rodando numa VM (ou um cluster leve tipo k3s) fisicamente em `southamerica-east1`, não no GKE de `us-central1`. Um flow marcado com uma tag `brasil` seria deployado com `work_pool_name="basedosdados-brasil"` em vez de `basedosdados`/`basedosdados-dev`; o worker que fica de olho nesse pool (em São Paulo) pega a run e executa o processo lá. Qualquer requisição HTTP que esse processo fizer já sai com IP brasileiro nativamente, sem `proxies=` em nenhum código de dataset.

### Como o worker dedicado funcionaria mecanicamente

O "worker" do Prefect não é o Prefect em si — é um processo simples que fica de olho (long-polling) na API (`https://prefect3.basedosdados.org/api`) perguntando se tem run agendada pro seu pool, e dispara a execução quando chega uma. O que muda é **como** ele dispara:

- **Process worker**: clona o repo (mesmo mecanismo já usado, `GitRepository` em `deploy_flow()`) e roda o flow como processo direto na VM, usando o Python que estiver instalado ali — frágil, porque exigiria replicar manualmente todas as dependências do `pipelines` (pandas, dbt, `basedosdados`, etc.) e manter isso sincronizado com `pyproject.toml`.
- **Docker worker**: sobe um container usando a mesma imagem que **já existe e já é usada em produção**. Confirmado em `pipelines/.github/workflows/build-docker-prefect3.yaml`: toda mudança no `Dockerfile` gera um build que sobe `gcr.io/<projeto>/pipelines:latest` (prod) / `:dev` (dev) e atualiza o `base_job_template.job_configuration.image` do work pool correspondente via `PATCH` na API do Prefect. Um Docker worker na VM de São Paulo reaproveitaria essa mesma imagem — sem precisar recriar o ambiente Python na mão. É o caminho mais natural pra uma VM única (não um cluster).
- **Kubernetes worker** (o que já roda hoje, em `us-central1`): só faz sentido com um cluster de verdade por trás — um k3s leve na própria VM, se fosse esse o caminho.

### Comparação de custo com a VM proxy

Nos dois cenários, uma VM em São Paulo é necessária de qualquer forma — a diferença é o que roda nela, e isso muda o custo bastante:

| | VM proxy (implementada) | VM com Docker worker dedicado |
|---|---|---|
| O que roda | Só Squid (repassa bytes, não processa nada) | Worker do Prefect + container da imagem `pipelines` rodando o flow inteiro (download, clean, upload, dbt) |
| Tamanho de máquina | `e2-small`/`e2-micro` | `e2-standard-4` (16GB) pra cima, dependendo de quais tabelas rodassem ali — referência real: `job_variables` de outros flows no repo já pedem de 4Gi a 16Gi de memória (`br_sfb_sicar` usa 12Gi) |
| Disco | Mínimo (só o SO) | Precisa de espaço pros arquivos temporários de download/clean — as tabelas grandes do CNPJ passam de alguns GB |
| Custo mensal estimado (compute, região SP) | ~$10-25/mês | ~$95-160/mês (estimativa grosseira, só a VM) |
| Mudança de código por dataset | Sim — `proxies=`/`BRASIL_PROXY_URL` explícito em cada dataset | Não — qualquer requisição do pod já sai brasileira |

**O fator que mais pesa não é o tamanho da máquina, é o modelo de cobrança.** Hoje, os Jobs do GKE só existem enquanto o flow roda de verdade — o node pool dinâmico escala pra 0 quando ocioso (`dynamic_pool_node_min = 0`, `terraform/terraform.tfvars`). Uma VM de Compute Engine não faz isso sozinha: fica ligada 24/7, e você paga o mês inteiro mesmo que o flow rode só alguns minutos por dia. Isso empilha em cima da diferença de tamanho de máquina — não é só "5-8x mais caro pelo tamanho", é isso **mais** o tempo ocioso que hoje não existe no modelo de Jobs efêmeros. Dá pra mitigar automatizando liga/desliga da VM perto dos horários agendados (Cloud Scheduler + Cloud Functions), mas isso é mais uma peça de automação pra manter, não é trivial.

Tráfego cross-region também existe nos dois casos (RF → VM em SP → destino final em `us-central1`, seja o pod do GKE via proxy, seja o bucket GCS via upload direto do worker), então não é um diferencial claro entre as opções — vale mencionar como custo real, mas não como motivo de decisão.

### Por que decidimos ir só com a VM proxy

A VM da opção atual serve como primeiro passo pros dois cenários — se o padrão "precisa de IP brasileiro" crescer além de `br_rf_cnpj`/`br_bd_execucao_estadual` (mais datasets, uso mais frequente), essa mesma VM (ou uma redimensionada) poderia evoluir pra rodar o worker dedicado, em vez de só o proxy.

Mas pro volume de hoje (1-2 datasets, uso pontual, nenhum flow contínuo de alta frequência precisando disso), a VM proxy vence em praticamente todo critério:
- **Custo**: dezenas de dólares/mês vs. uma a duas centenas.
- **Superfície de segurança**: a VM proxy só guarda a própria senha do Squid, inventada por nós. Um worker dedicado precisaria de credencial de verdade pra rodar os flows — service account do GCP com permissão de GCS/BigQuery, token do Vault — uma superfície de risco bem maior se a VM for comprometida.
- **Manutenção**: um serviço simples (Squid) vs. manter um worker do Prefect + a sincronia com a imagem Docker de produção rodando fora do GKE.
- **Reversibilidade**: se o proxy parar de ser necessário, é só desligar a VM e reverter o código — nada em produção depende dela além das chamadas de rede explicitamente configuradas.

**Status**: ideia do work pool dedicado registrada, não implementada. Revisitar se o número de fontes bloqueadas por geolocalização crescer o suficiente pra justificar o custo e a complexidade extra — não é o caso agora.
