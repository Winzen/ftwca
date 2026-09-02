https://github.com/jaegertracing/jaeger-operator#jager-v2-operator
https://github.com/open-telemetry/opentelemetry-proto/blob/main/docs/specification.md#otlpgrpc
https://github.com/open-telemetry/opentelemetry-python/tree/main/docs/examples/django
https://uptrace.dev/guides/opentelemetry-django
https://opentelemetry.io/blog/2024/getting-started-with-otelsql/
https://github.com/open-telemetry/opentelemetry-proto/blob/main/docs/specification.md#otlpgrpc
https://github.com/open-telemetry/opentelemetry-collector/blob/main/receiver/otlpreceiver/README.md


### Variaveis de ambientes util

- **`OTEL_SERVICE_NAME`**: define o nome do serviço que aparecerá nas ferramentas de observabilidade (ex: `django-api`).
    
- **`OTEL_EXPORTER_OTLP_ENDPOINT`**: endereço do **OpenTelemetry Collector** responsável por receber os traces (ex: `otel-collector:4317`).
    
- **`OTEL_EXPORTER_OTLP_PROTOCOL`**: protocolo utilizado para envio dos dados ao collector, normalmente `grpc`.
    
- **`OTEL_EXPORTER_OTLP_INSECURE`**: permite conexão sem TLS com o collector, comum em ambientes locais ou Docker.
    
- **`OTEL_TRACES_EXPORTER`**: define para onde os traces serão exportados, como `otlp` (collector) ou `console` (logs da aplicação).
    
- **`OTEL_TRACES_SAMPLER`**: controla a estratégia de coleta de traces, por exemplo `always_on` para coletar todos.
    
- **`OTEL_INSTRUMENTATION_DBAPI_CAPTURE_STATEMENT`**: habilita a captura das queries SQL executadas pela aplicação.
    
- **`OTEL_INSTRUMENTATION_DJANGO_INSTRUMENT_DBAPI`**: ativa a instrumentação automática do Django ORM para registrar operações de banco nos traces.
```
api: build: context: . dockerfile: Dockerfile environment: - OTEL_SERVICE_NAME=django-api - OTEL_EXPORTER_OTLP_ENDPOINT=otel-collector:4317 - OTEL_EXPORTER_OTLP_PROTOCOL=grpc - OTEL_TRACES_EXPORTER=console,otlp - OTEL_EXPORTER_OTLP_INSECURE=true - OTEL_INSTRUMENTATION_DBAPI_CAPTURE_STATEMENT=true - OTEL_INSTRUMENTATION_DJANGO_INSTRUMENT_DBAPI=true - OTEL_TRACES_SAMPLER=always_on
```
### otel-collector-config.yaml
```
# otel-collector-config.yaml

receivers:

  otlp:

    protocols:

      grpc:    # Recebe traces do Django API via gRPC (porta 4317)

      http:    # Recebe traces via HTTP (porta 4318, opcional)

  

exporters:

  zipkin:

    endpoint: "http://jaeger:9411/api/v2/spans"  # Porta Zipkin HTTP do Jaeger

  logging:

    loglevel: debug

  

processors:

  batch:

    timeout: 10s

    send_batch_size: 50

  

service:

  pipelines:

    traces:

      receivers: [otlp]

      processors: [batch]

      exporters: [zipkin, logging]

    metrics:

      receivers: [otlp]

      processors: [batch]

      exporters: [logging]
```

```


Perfeito! Aqui está um **resumo organizado da sua conversa**, com todas as instruções e arquivos prontos para usar como referência rápida. Vou separar por seções para ficar bem legível e fácil de salvar.

---

## 1. **Objetivo**

- Instrumentar Django + GraphQL + PostgreSQL usando **OpenTelemetry**.
    
- Enviar **traces HTTP e queries SQL** para **Jaeger**.
    
- Visualizar no Jaeger todas as requisições, incluindo queries de banco, para otimização.
    

---

## 2. **Docker Setup (docker-compose.yaml)**

```yaml
version: '3.8'
services:
  index:
    image: elasticsearch:7.17.3
    container_name: index
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports: [9200:9200, 9300:9300]
    volumes: [esdata:/usr/share/elasticsearch/data]
    healthcheck:
      test: [CMD-SHELL, curl -s http://localhost:9200 >/dev/null || exit 1]
      interval: 1m
      timeout: 20s
      retries: 5
      start_period: 1m

  queue:
    image: redis:6.0
    container_name: queue
    ports: [6379:6379]
    healthcheck:
      test: [CMD, redis-cli, ping]
      interval: 10s
      timeout: 5s
      retries: 3

  database:
    image: postgres:14
    container_name: database
    environment: [POSTGRES_USER=postgres, POSTGRES_HOST_AUTH_METHOD=trust]
    env_file: [.env]
    ports: [5432:5432]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: [CMD, pg_isready, -U, postgres]
      interval: 1m
      timeout: 30s
      retries: 5
      start_period: 1m
    restart: unless-stopped

  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - OTEL_SERVICE_NAME=django-api
      - OTEL_EXPORTER_OTLP_ENDPOINT=otel-collector:4317
      - OTEL_EXPORTER_OTLP_PROTOCOL=grpc
      - OTEL_TRACES_EXPORTER=otlp
    container_name: api
    command: [poetry, run, /app/start-server-dev.sh]
    ports:
      - 8000:8000  # API port
      - 5678:5678  # Debug port
    volumes: [.:/app, $HOME/.basedosdados/credentials:/app/credentials:ro]
    networks: [default, api_network]
    depends_on:
      index:
        condition: service_healthy
      queue:
        condition: service_healthy
      database:
        condition: service_healthy
      otel-collector:
        condition: service_started
      jaeger:
        condition: service_started
    healthcheck:
      test: [CMD, curl, -f, http://localhost:8000/healthcheck/]
      interval: 1m
      timeout: 30s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    container_name: otel-collector
    command: ["--config=/otel_collector/otel-config.yaml"]
    volumes:
      - ./otel_collector/otel-collector-config.yaml:/otel_collector/otel-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
    networks: [default, api_network]
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:1.43
    container_name: jaeger
    ports:
      - "16686:16686" # UI
      - "14250:14250" # gRPC
      - "14268:14268" # HTTP
      - "9411:9411"   # Zipkin
    restart: unless-stopped
    networks: [default, api_network]

volumes:
  esdata:
  pgdata:

networks:
  api_network:
    name: api_network
```

---

## 3. **OpenTelemetry Collector (otel-collector-config.yaml)**

```yaml
receivers:
  otlp:
    protocols:
      grpc:   # Porta 4317 para gRPC
      http:   # Porta 4318 para HTTP (opcional)

exporters:
  otlphttp:
    endpoint: "http://jaeger:14268/api/traces"  # Jaeger HTTP collector
  logging:
    loglevel: debug

processors:
  batch:
    timeout: 10s
    send_batch_size: 50

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp, logging]
```

> ⚠️ **Não use “jaeger” diretamente como exporter**, pois na versão que você estava usando do collector não é reconhecido. Use `otlphttp` apontando para a porta HTTP do Jaeger (`14268`).

---

## 4. **start-server-dev.sh**

```bash
#!/usr/bin/env bash

echo "> Collecting static files"
(cd /app; python manage.py collectstatic --no-input --settings=backend.settings.base)

echo "> Making migrations"
(cd /app; python manage.py makemigrations)

echo "> Applying migrations"
(cd /app; python manage.py migrate)

echo "> Installing debugpy"
pip install debugpy

echo "> Creating superuser"
if [ -n "$DJANGO_SUPERUSER_USERNAME" ] && [ -n "$DJANGO_SUPERUSER_PASSWORD" ] ; then
    (cd /app; python manage.py createsuperuser --no-input)
fi

echo "> Running Huey"
(cd /app; python manage.py run_huey &)

echo "> Running server in development mode with OpenTelemetry"
(cd /app; poetry run opentelemetry-instrument python manage.py runserver 0.0.0.0:8000 --noreload)
```

---

## 5. **Django Settings (instrumentação completa)**

No **topo do `settings.py` ou `settings/dev.py`**, antes de qualquer acesso ao DB:

```python
from opentelemetry.instrumentation.django import DjangoInstrumentor
from opentelemetry.instrumentation.dbapi import wrap_connect
import psycopg2

# Instrumenta HTTP + ORM
DjangoInstrumentor().instrument(
    is_sql_commentor_enabled=True,
    instrument_dbapi=True
)

# Instrumenta psycopg2 DBAPI diretamente
wrap_connect(
    psycopg2,
    connect_method_name="connect",
    database_system="postgresql",
    service="django-db",
    capture_statement=True
)
```

> Isso garante que **todas as queries do Django ORM apareçam no Jaeger**.

---

## 6. **PostgreSQL (DATABASES)**

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
        "NAME": getenv("DB_NAME"),
        "USER": getenv("DB_USER"),
        "PASSWORD": getenv("DB_PASSWORD"),
        "HOST": getenv("DB_HOST"),
        "PORT": getenv("DB_PORT"),
    }
}
```

---

## 7. **Pacotes Python necessários**

```text
opentelemetry-api                        1.40.0
opentelemetry-sdk                        1.40.0
opentelemetry-instrumentation            0.61b0
opentelemetry-instrumentation-django     0.61b0
opentelemetry-instrumentation-dbapi      0.61b0
opentelemetry-instrumentation-psycopg2   0.61b0
opentelemetry-exporter-otlp              1.40.0
opentelemetry-exporter-otlp-proto-grpc   1.40.0
opentelemetry-exporter-otlp-proto-http   1.40.0
```

---

## 8. **Como verificar no Jaeger**

1. Acesse: [http://localhost:16686](http://localhost:16686)
    
2. Procure pelo **serviço `django-api`**.
    
3. Você verá spans do Django HTTP **e todos os queries do PostgreSQL** dentro do mesmo trace.
    

> Cada request HTTP agora terá sub-spans com o SQL executado, incluindo SELECT, INSERT, UPDATE, DELETE.

---

Se você quiser, posso te enviar **uma versão final do `settings.py` completo pronta para dev**, já configurada para Jaeger + PostgreSQL + GraphQL + Django, que você só coloca no container e já começa a rastrear tudo.

Quer que eu faça isso?