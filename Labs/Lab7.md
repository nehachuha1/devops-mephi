# Технологии мониторинга Prometheus и Grafana

## Конфигурация

### Node exporter

Повторить для машины, на которой стоит Gitlab и для всех узлов Kubernetes:
  - Установить пакет `prometheus-node-exporter` (`sudo apt install prometheus-node-exporter`)
  - Проверить установку командой `curl http://localhost:9100/metrics`

### Grafana

- В файл `docker-compose.yml` с Gitlab добавить сервис `grafana`:
  - Использовать образ `grafana/grafana-oss:10.2.0`
  - Указать следующие переменные среды:
    ```yaml
    GF_ANALYTICS_REPORING_ENABLED: "false"
    GF_SERVER_DOMAIN: localhost:3000
    GF_SERVER_ROOT_URL: http://localhost:3000
    ```
  - Добавить named volume: `grafana:/var/lib/grafana`
  - Пробросить порт `3000`

### Pushgateway

- В файл `docker-compose.yml` с Gitlab добавить сервис `pushgateway`:
  - Использовать образ `prom/pushgateway:v1.4.2`
  - В `command` указать:
    ```yaml
    - "--persistence.file=/pushgateway/data"
    - "--persistence.interval=3m"
    - "--web.enable-admin-api"
    - "--web.external-url=http://localhost:9091"
    ```
  - Добавить named volume: `pushgateway:/pushgateway`
  - Пробросить порт `9091`

### Prometheus

- Создать файл `prometheus.yml`:
  ```yaml
  global:
    scrape_interval: 15s
    evaluation_interval: 15s

  scrape_configs: []
  ```
- В файл `docker-compose.yml` с Gitlab добавить сервис `prometheus`:
  - Использовать образ `prom/prometheus:v2.47.2`
  - В `command` указать:
    ```yaml
    - "--config.file=/etc/prometheus/prometheus.yml"
    - "--storage.tsdb.path=/prometheus"
    - "--storage.tsdb.retention.time=30d"
    - "--web.console.libraries=/usr/share/prometheus/console_libraries"
    - "--web.console.templates=/usr/share/prometheus/consoles"
    - "--web.enable-admin-api"
    - "--web.external-url=http://localhost:9090"
    ```
  - Прописать параметр `extra_hosts`, чтобы у контейнера был сетевой доступ к хосту по домену `host.docker.internal`:
    ```yaml
    extra_hosts:
      - host.docker.internal:host-gateway
    ```
  - Смонтировать созданный файл конфигурации в `/etc/prometheus/prometheus.yml`:
    ```
    ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ```
  - Добавить named volume: `prometheus:/prometheus`
  - Пробросить порт `9090`
- В файле `./prometheus.yml` в `scrape_configs` прописать источники метрик:
  - Сбор метрик с node exporter:
    ```yaml
    - job_name: node
      static_configs:
        - targets:
          - host.docker.internal:9100
          - ksnode1.local:9100
          - ksnode2.local:9100
      metrics_path: /metrics
    ```
  - Сбор метрик с prometheus:
    ```yaml
    - job_name: prometheus
      static_configs:
        - targets: [localhost:9090]
      metrics_path: /metrics
    ```
  - Аналогично прописать сбор метрик для:
    - Grafana - `grafana:3000`
    - Pushgateway - `pushgateway:9091`
    - Приложения todo - `todo2.local:80`
  - Для сбора метрик с Gitlab указать:
    ```yaml
    - job_name: gitlab
      static_configs:
        - targets: [gitlab:8929]
      scheme: http
      metrics_path: /-/metrics
      params:
        token: ["<gitlab-token>"]
    ```
    `gitlab-token` можно получить на Gitlab -> Admin Area -> Monitoring -> Health Check -> Access Token

## Запуск

- Запустить все приложения `docker compose up -d`
- Убедиться, что все сервисы запущены (`docker compose ps`)
- Открыть Prometheus в браузере (http://localhost:9090)
- В Prometheus открыть страницу Status -> Targets
- Убедиться, что все источники метрик работают
- Открыть Grafana (http://localhost:3000) (логин - `admin`, пароль - `admin`)
- В Grafana добавить Prometheus, как источник данных:
  - Открыть Connections -> Data sources -> Add data source -> Prometheus
  - В Prometheus server URL указать http://prometheus:9090
  - Нажать Save & test

## Создание Grafana Dashboard

### Метрики узлов

- В Grafana открыть Dashboards -> Create Dashboard -> Import dashboard
- Указать Dashbopard URL: https://grafana.com/grafana/dashboards/1860-node-exporter-full/
- Load -> выбрать Prometheus -> Import
- Проанализировать метрики на созданном Dashboard для всех машин

### Метрики Gitlab

- Создать новый Dashboard (Dashboards -> Create Dashboard)
- Добавить новую панель (Add visualization -> Prometheus):
  - Title: `Gitlab RPS`
  - Metric: `http_requests_total`
  - Label filters:
    - Label: `job`
    - Value: `gitlab`
  - Operations -> Range functions -> Rate:
    - Range: `$__rate_interval`
  - Operations -> Aggregations -> Sum:
    - Label: `instance`
  - Run queries
  - Проверить график
  - Можно поиграться с различными настройками графика, цветами
  - Нажать Apply
- Сохранить Dashboard

### Метрики приложения

- По аналогии добавить еще одну панель для приложения todo2.local:
  - Использовать метрику `api_requests_total`
- Сохранить Dashboard