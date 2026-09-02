```
{

  "annotations": {

    "list": [

      {

        "builtIn": 1,

        "enable": true,

        "hide": true,

        "iconColor": "rgba(0, 211, 255, 1)",

        "name": "Annotations & Alerts",

        "type": "dashboard"

      }

    ]

  },

  "editable": true,

  "fiscalYearStartMonth": 0,

  "graphTooltip": 0,

  "id": 1532,

  "links": [],

  "liveNow": false,

  "panels": [

    {

      "datasource": {

        "type": "loki",

        "uid": "ZG5SePISk"

      },

      "fieldConfig": {

        "defaults": {

          "color": {

            "mode": "palette-classic"

          },

          "custom": {

            "axisBorderShow": false,

            "axisCenteredZero": false,

            "axisColorMode": "text",

            "axisLabel": "",

            "axisPlacement": "auto",

            "barAlignment": 0,

            "drawStyle": "line",

            "fillOpacity": 0,

            "gradientMode": "none",

            "hideFrom": {

              "legend": false,

              "tooltip": false,

              "viz": false

            },

            "insertNulls": false,

            "lineInterpolation": "linear",

            "lineWidth": 1,

            "pointSize": 5,

            "scaleDistribution": {

              "type": "linear"

            },

            "showPoints": "auto",

            "spanNulls": false,

            "stacking": {

              "group": "A",

              "mode": "none"

            },

            "thresholdsStyle": {

              "mode": "off"

            }

          },

          "mappings": [],

          "thresholds": {

            "mode": "absolute",

            "steps": [

              {

                "color": "green",

                "value": null

              },

              {

                "color": "red",

                "value": 80

              }

            ]

          }

        },

        "overrides": []

      },

      "gridPos": {

        "h": 9,

        "w": 12,

        "x": 0,

        "y": 0

      },

      "id": 2,

      "options": {

        "legend": {

          "calcs": [],

          "displayMode": "list",

          "placement": "bottom",

          "showLegend": true

        },

        "tooltip": {

          "mode": "single",

          "sort": "none"

        }

      },

      "targets": [

        {

          "datasource": {

            "type": "loki",

            "uid": "ZG5SePISk"

          },

          "editorMode": "code",

          "expr": "sum(count_over_time({app=\"$environment\"} [$__interval]))",

          "queryType": "range",

          "refId": "A"

        }

      ],

      "title": "Requisições",

      "type": "timeseries"

    },

    {

      "datasource": {

        "type": "loki",

        "uid": "ZG5SePISk"

      },

      "gridPos": {

        "h": 17,

        "w": 12,

        "x": 12,

        "y": 0

      },

      "id": 8,

      "options": {

        "dedupStrategy": "none",

        "enableLogDetails": true,

        "prettifyLogMessage": false,

        "showCommonLabels": false,

        "showLabels": false,

        "showTime": false,

        "sortOrder": "Descending",

        "wrapLogMessage": false

      },

      "targets": [

        {

          "datasource": {

            "type": "loki",

            "uid": "ZG5SePISk"

          },

          "editorMode": "code",

          "expr": "{app=\"$environment\"} != `huey` != `elasticsearch` | json | __error__=``",

          "queryType": "range",

          "refId": "A"

        }

      ],

      "title": "Logs",

      "type": "logs"

    },

    {

      "datasource": {

        "type": "loki",

        "uid": "ZG5SePISk"

      },

      "gridPos": {

        "h": 8,

        "w": 12,

        "x": 0,

        "y": 9

      },

      "id": 4,

      "options": {

        "dedupStrategy": "none",

        "enableLogDetails": true,

        "prettifyLogMessage": false,

        "showCommonLabels": false,

        "showLabels": false,

        "showTime": false,

        "sortOrder": "Descending",

        "wrapLogMessage": false

      },

      "targets": [

        {

          "datasource": {

            "type": "loki",

            "uid": "ZG5SePISk"

          },

          "editorMode": "code",

          "expr": "{app=\"api-prod\"} |= \"huey\"",

          "queryType": "range",

          "refId": "A"

        }

      ],

      "title": "Tarefas (Huey)",

      "type": "logs"

    },

    {

      "datasource": {

        "type": "loki",

        "uid": "ZG5SePISk"

      },

      "gridPos": {

        "h": 5,

        "w": 24,

        "x": 0,

        "y": 17

      },

      "id": 6,

      "options": {

        "dedupStrategy": "none",

        "enableLogDetails": true,

        "prettifyLogMessage": false,

        "showCommonLabels": false,

        "showLabels": false,

        "showTime": false,

        "sortOrder": "Descending",

        "wrapLogMessage": false

      },

      "targets": [

        {

          "datasource": {

            "type": "loki",

            "uid": "ZG5SePISk"

          },

          "editorMode": "code",

          "expr": "{app=\"$environment\"} |= \"elasticsearch\"",

          "queryType": "range",

          "refId": "A"

        }

      ],

      "title": "Busca (Elasticsearch)",

      "type": "logs"

    }

  ],

  "schemaVersion": 39,

  "tags": [],

  "templating": {

    "list": [

      {

        "current": {

          "selected": false,

          "text": "prd",

          "value": "api-prod"

        },

        "hide": 0,

        "includeAll": false,

        "multi": false,

        "name": "environment",

        "options": [

          {

            "selected": true,

            "text": "prd",

            "value": "api-prod"

          },

          {

            "selected": false,

            "text": "stg",

            "value": "api-staging"

          },

          {

            "selected": false,

            "text": "dev",

            "value": "api-development"

          }

        ],

        "query": "prd : api-prod,stg : api-staging,dev : api-development",

        "queryValue": "",

        "skipUrlSync": false,

        "type": "custom"

      }

    ]

  },

  "time": {

    "from": "now-1h",

    "to": "now"

  },

  "timepicker": {},

  "timezone": "",

  "title": "Backend",

  "uid": "dfehik12dezggf",

  "version": 1,

  "weekStart": ""

}
```




---

## Reinicialização Completa do PVC e do Pod

Após a confirmação de que o disco do Loki estava 100% ocupado e que o pod permanecia em **CrashLoopBackOff**, optamos por realizar um reset completo do armazenamento persistente.

Como não havia necessidade de preservar os logs históricos, a solução escolhida foi remover o volume persistente e permitir que o Kubernetes recriasse o ambiente limpo automaticamente.

### Remoção do PVC

Executamos a exclusão do PersistentVolumeClaim:

```bash
kubectl delete pvc storage-loki-0 -n observability
```

Essa ação removeu o volume associado ao pod `loki-0`, liberando completamente o armazenamento que estava saturado.

Como o Loki está configurado via StatefulSet, o Kubernetes automaticamente:

- Criou um novo PVC com o mesmo nome
    
- Provisionou um novo volume vazio
    
- Recriou o pod com o storage limpo
    

---

### Ajuste Temporário do StatefulSet

Durante o processo de troubleshooting, foram realizadas alterações temporárias no StatefulSet.

Após a normalização do ambiente, executamos:

```bash
kubectl edit statefulset loki -n observability
```

O StatefulSet foi restaurado para sua configuração original, garantindo que não permanecessem ajustes de debug ou alterações emergenciais.

---

### Atualização do ConfigMap

Em seguida, realizamos a edição do ConfigMap do Loki para aplicar a política de retenção e evitar reincidência do problema:

```bash
kubectl edit configmap loki -n observability
```

Foi adicionada a configuração:

```yaml
limits_config:
  retention_period: 168h
```

Essa configuração garante que logs com mais de 7 dias sejam automaticamente removidos, prevenindo o crescimento indefinido do volume.

---

### Escalar o StatefulSet novamente para 1 réplica

Após concluir os ajustes, reativamos o Loki:
```
kubectl scale statefulset loki --replicas=1 -n observability
```


O Kubernetes então:

- Criou um novo PVC `storage-loki-0`
    
- Provisionou um novo volume vazio
    
- Recriou o pod `loki-0`
    
- Inicializou o Loki com storage limpo