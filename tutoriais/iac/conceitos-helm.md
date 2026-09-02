# Conceitos: Helm, Charts e Values

## O que é Helm?

Helm é o gerenciador de pacotes do Kubernetes — pensa nele como um `apt` ou `pip`, mas para aplicações K8s.

Em vez de você escrever manualmente dezenas de arquivos YAML (Deployment, Service, ServiceAccount, ConfigMap...), o Helm empacota tudo isso num **chart** e você só precisa dizer "instala isso aqui com essas configurações".

---

## O que é um Chart?

Um **chart** é o pacote em si — é um conjunto de templates YAML parametrizados que descrevem todos os recursos Kubernetes necessários para rodar uma aplicação.

Estrutura típica de um chart:
```
prefect-server/
├── Chart.yaml          # metadados: nome, versão, descrição
├── values.yaml         # valores padrão de todas as variáveis
└── templates/          # os YAMLs com variáveis ({{ .Values.algo }})
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── ...
```

O chart do Prefect 3 está hospedado publicamente no GitHub:
📦 https://github.com/PrefectHQ/prefect-helm/tree/main/charts/prefect-server

---

## O que é o values.yaml?

O `values.yaml` é onde você **sobrescreve os padrões** do chart para o seu caso de uso.

Funciona assim:
- O chart tem um `values.yaml` padrão com todas as configurações possíveis
- Você cria o **seu** `values.yaml` só com o que quer mudar
- Na hora do `helm upgrade --install`, o Helm mescla os dois

**values.yaml padrão do chart** (definido pelo PrefectHQ):
```yaml
postgresql:
  enabled: true      # por padrão sobe um postgres interno
server:
  replicaCount: 1
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
```

**nosso values.yaml** (`k8s/prefect3/chart/values.yaml`):
```yaml
postgresql:
  enabled: false     # desligamos — usamos o Cloud SQL externo
server:
  resources:
    requests:
      cpu: 100m      # reduzimos para o ambiente de teste
      memory: 128Mi
```

O resultado final é a mesclagem dos dois — o que não declaramos no nosso, vem do padrão do chart.

---

## De onde vieram os templates?

O `values.yaml` padrão do chart do Prefect 3 foi buscado diretamente do repositório oficial:

```
https://raw.githubusercontent.com/PrefectHQ/prefect-helm/main/charts/prefect-server/values.yaml
```

O arquivo tem ~600 linhas com todos os parâmetros possíveis comentados. A partir dele, identificamos o que precisávamos ajustar para o nosso ambiente:

| Parâmetro padrão | Nosso valor | Motivo |
|---|---|---|
| `postgresql.enabled: true` | `false` | Usamos Cloud SQL |
| `ingress.enabled: false` | mantido `false` | Temos nosso próprio ingress.yaml |
| `secret.create: true` | `false` | Usamos sealed secret próprio |
| `server.uiConfig.prefectUiApiUrl: localhost` | `https://prefect3.basedosdados.org/api` | URL pública real |
| `server.resources` (500m/512Mi) | 100m/128Mi | Ambiente de teste com recursos limitados |

---

## Fluxo completo com Helm

```
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
          ↓
  Baixa o chart (templates + values padrão)

helm upgrade --install prefect-server -n prefect3 prefecthq/prefect-server -f values.yaml
          ↓
  Mescla values padrão + nosso values.yaml
          ↓
  Renderiza os templates YAML com os valores finais
          ↓
  Aplica todos os recursos no namespace prefect3 do GKE
```
