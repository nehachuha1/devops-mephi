# Сбор логов при помощи Loki

## Конфигурация

### Loki

- На сервере создать файл конфигурации `loki-config.yaml`:
  ```yaml
  auth_enabled: false

  server:
    http_listen_port: 3100

  common:
    ring:
      instance_addr: 127.0.0.1
      kvstore:
        store: inmemory
    replication_factor: 1
    path_prefix: /tmp/loki

  schema_config:
    configs:
    - from: 2020-05-15
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

  storage_config:
    filesystem:
      directory: /tmp/loki/chunks
  ```

- В файл `docker-compose.yml` с Grafana и Gitlab добавить сервис loki:
  - Образ: `grafana/loki:3.0.0`
  - Порт: `3100`
  - Добавить `command`: `-config.file=/mnt/config/loki-config.yaml`
  - Смонтировать конфиг `/mnt/config/loki-config.yaml` на `loki-config.yaml` на хосте
  - Смонтировать папку `/tmp/loki` конейнера в named volume

### Promtail

  - На сервере создать файл конфигурации `promtail-config.yaml`:
    ```yaml
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /tmp/positions.yaml

    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
      - job_name: system
        static_configs:
          - targets:
              - localhost
            labels:
              job: varlogs
              __path__: /var/log/*log

      - job_name: docker-json
        static_configs:
          - targets:
              - localhost
            labels:
              job: docker_logs
              __path__: /var/lib/docker/containers/*/*-json.log
        pipeline_stages:
          - json:
              expressions:
                log: log
                stream: stream
                time: time
                tag: attrs.tag
                compose_project: attrs."com.docker.compose.project"
                compose_service: attrs."com.docker.compose.service"
          - regex:
              expression: "^/var/lib/docker/containers/(?P<container_id>.{12}).+/.+-json.log$"
              source: filename
          - regex:
              expression: "(\\d+-\\d+-\\dT\\d+:\\d+:\\d+\\.\\d+Z)\\s+(\\w+)\\s+(.*?)"
              labels:
                timestamp: "${1}"
                level: "${2}"
          - template:
              source: level
              template: '{{ regexReplaceAllLiteral "DETAIL|DEBUG|INFO|NOTICE|LOG" .level "info" }}'
          - timestamp:
              format: RFC3339Nano
              source: time
          - labels:
              stream:
              container_id:
              tag:
              # docker compose
              compose_project:
              compose_service:
          - output:
              source: log

      - job_name: docker-txt
        static_configs:
          - targets:
              - localhost
            labels:
              job: docker_logs
              __path__: /var/lib/docker/containers/**/*.log
    ```

- В файл `docker-compose.yml` с Grafana и Gitlab добавить сервис promtail:
  - Образ: `grafana/promtail:3.0.0`
  - Добавить `command`: `-config.file=/mnt/config/promtail-config.yaml`
  - Смонтировать файл конфигурации `/mnt/config/promtail-config.yaml` на `promtail-config.yaml` на хосте
  - Смонтировать хостовые папки `/var/logs` и `/var/lib/docker/containers` в контейнер
  - Смонтировать папку `/tmp` конейнера в named volume
  - Добавить сервис `loki` в зависимости

## Запуск

- Запустить все сервисы `docker compose up -d`
- Убедиться, что все сервисы запущены
- В Grafana добавить Loki, как источник данных:
  - Открыть Connections -> Data sources -> Add data source -> Loki
  - В URL указать http://loki:3100
  - Нажать Save & test

## Отображение логов

- Создать новый Dashboard (Dashboards -> Create Dashboard)
- Добавить новую панель (Add visualization -> Loki)
- Изменить тип панели с Time series на Logs
- Отфильтровать логи на ваше усмотрение