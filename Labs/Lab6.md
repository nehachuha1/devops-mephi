# Шаблонизатор развертывания Helm

## Написание шаблонов

- В папке с проектом сгенирировать папку `todo` с Helm Charts: `helm create todo`
- Изучить сгенерированные файлы
- Переименовать папку `todo/templates` в `todo/examples`
- Создать папку `todo/templates`
- В дальнейшем использовать шаблоны из папки `todo/examples` в качестве примера
- Отредактировать файл `todo/Chart.yaml`:
  - Изменить `name` в файле `todo/Chart.yaml` на `todo`
  - Изменить `description`
- Заменить содержимое файла `todo/values.yaml` (изменить `<username>` на свой `username` в Gitlab) на:
  ```yaml
  replicas: 2

  image:
    registry: gitlab.local:5005
    name: <username>/devops-todo
    tag: main
    pullPolicy: IfNotPresent
    pullSecret: ""

  nameOverride: ""

  ingress:
    className: ""
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: web
    hosts:
      - host: todo2.local
        path: /
        pathType: Prefix

  postgresql:
    enabled: true
    auth:
      postgresPassword: ""
      username: todo
      password: ""
      database: todo
    image:
      tag: "15.4.0"
  ```

- В папке `todo/templates` создать файл `_helpers.tpl`:
  ```go
  {{/*
  Expand the name of the chart.
  */}}
  {{- define "todo.name" -}}
  {{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
  {{- end }}

  {{/*
  Create chart name and version as used by the chart label.
  */}}
  {{- define "todo.chart" -}}
  {{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
  {{- end }}

  {{/*
  Selector labels.
  */}}
  {{- define "todo.selectorLabels" -}}
  app: {{ include "todo.name" . }}
  {{- end }}

  {{/*
  Common labels.
  */}}
  {{- define "todo.labels" -}}
  helm.sh/chart: {{ include "todo.chart" . }}
  {{ include "todo.selectorLabels" . }}
  {{- end }}

  {{/*
  Image.
  */}}
  {{- define "todo.image" -}}
  {{- printf "%s/%s:%s" .Values.image.registry .Values.image.name .Values.image.tag }}
  {{- end }}
  ```

- В папке `todo/templates` создать файлы:
  - `deployment.yaml`
  - `ingress.yaml`
  - `service.yaml`

- В каждый из созданных файлов скопировать соответсвующую секцию из файла `.k8s/deployment.yaml`
- Создать файл `todo/templates/configmap.yaml`:
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: {{ include "todo.name" . }}
    labels:
      {{- include "todo.labels" . | nindent 4 }}
  data:
    {{ with .Values.postgresql.auth }}
    DATABASE_URL: {{ printf "postgresql://%s:%s@todo-postgresql:5432/%s?schema=public" .username .password .database | quote }}
    {{- end }}
  ```
- По аналогии шаблонизировать остальные файлы в папке `todo/templates` подставляя туда переменные из файла `values.yaml`. Рекомендации:
  - Ориентироваться на примеры по-умолчанию из папки `todo/examples`
  - Не забывать использовать хелперы из файла `todo/templates/_helpers.tpl`
  - Из всех файлов убрать `namespace`, т.к. Helm подставляет его автоматически
  - При объявлении ingress использовать цикл для объявления хостов:
    ```yaml
    {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "todo.name" $ }}
                port:
                  number: 3000
    {{- end }}
    ```
  - Для проверки конфигурации использовать команду:
    ```
    helm install todo ./todo --debug --dry-run
    ```
    Она выведет сгенерированные конфигурации Kubernetes

## Добавление PostgreSQL

- В файл `todo/Chart.yaml` добавить секцию `dependencies` и прописать в ней PostgreSQL:
  ```yaml
  dependencies:
    - name: postgresql
      version: "13.1.5"
      repository: https://charts.bitnami.com/bitnami
      condition: postgresql.enabled
  ```
- Собрать зависимости командой `helm dependency build ./todo`

## Деплой приложения

- Создать новый namespace в Kubernetes: `kubectl create namespace task6`
- Скопировать ключ доступа к Docker Registry из namespace `task5` в новый `namespace`:
  ```bash
  kubectl -n task5 get secret todo-regcred --output=yaml | sed 's/namespace: .*/namespace: task6/' | kubectl apply -f -
  ```
- На хосте отредактировать файл `/etc/hosts` (`C:\Windows\System32\drivers\etc\hosts`) и добавить туда домен `todo2.local`, указывающий на мастер узел Kubernetes
- Установить Helm конфигурацию (в команде заменить пароли):
  ```bash
  helm -n task6 install \
    --set "postgresql.auth.postgresPassword=MASTERPASS" \
    --set "postgresql.auth.password=TODOPASS" \
    --set "image.pullSecret=todo-regcred" \
    todo ./todo
  ```
- Дождаться успешного запуска подов: `kubectl -n task6 get pods -o wide`
- В случае ошибок, обновить конфигурацию можно следующей командой (в команде заменить пароли):
  ```bash
  helm -n task6 upgrade \
    --set "postgresql.auth.postgresPassword=MASTERPASS" \
    --set "postgresql.auth.password=TODOPASS" \
    --set "image.pullSecret=todo-regcred" \
    todo ./todo
  ```
- Проверить доступность сервиса на http://todo2.local

## Настройка Gitlab CI для использования Helm

- Изменить/создать переменные `DB_ROOT_PASSWORD` и `DB_PASSWORD` в Gitlab
- Изменить задачу `deploy` в `.gitlab-ci.yml`:
  - Изменить образ на `alpine/helm:3.13.1`
  - В начале выполнения задачи:
    ```bash
    mkdir -p ~/.kube
    cp "$KUBE_CONFIG" ~/.kube/config
    chmod 600 ~/.kube/config
    ```
  - Во время выполнения задачи:
    ```bash
    helm -n task6 upgrade \
      --set "postgresql.auth.postgresPassword=$DB_ROOT_PASSWORD" \
      --set "postgresql.auth.password=$DB_PASSWORD" \
      --set "image.pullSecret=todo-regcred" \
      todo ./todo
    ```
- Для того, что проверить обновление приложения изменить файл `src/app/page.tsx`:
  - Добавить тег `<h1></h1>` с любым текстом внутри
  ```tsx
  "use client"

  import { TodoList } from "@/components/TodoList"
  import type { ReactNode } from "react"

  export default function Home(): ReactNode {
    return (
      <main className="min-h-screen sm:py-12">
        <h1>Hello world task6!</h1>
        <TodoList />
      </main>
    )
  }
  ```
- Запушить изменения в ветку `main`
- Дождаться успешного завершения pipeline-а
- Проверить состояние подов: `kubectl -n task5 get pods -o wide`
- Проверить внесенные в приложение изменения: http://todo2.local