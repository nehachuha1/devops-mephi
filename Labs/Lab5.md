# Деплой приложения в Kubernetes

## Подготовка

### Объединение Gitlab и кластера Kubernetes в сеть

#### Gitlab находится на VirtualBox

- В настройках виртуальной машины Gitlab в Сети включить Адаптер 2:
  - Тип подключения: Виртуальный адаптер хоста
  - Имя: выбрать созданную выше Host-only сеть

#### Gitlab находится в WSL

- Пробросить порты в Host-only сеть:
  ```powershell
  netsh interface portproxy add v4tov4 listenport=8929 listenaddress=192.168.12.1 connectport=8929 connectaddress=<WSL eth0 address>
  netsh interface portproxy add v4tov4 listenport=5005 listenaddress=192.168.12.1 connectport=5005 connectaddress=<WSL eth0 address>
  netsh interface portproxy add v4tov4 listenport=2224 listenaddress=192.168.12.1 connectport=2224 connectaddress=<WSL eth0 address>
  ```
- Отключить Windows Firewall для `VirtualBox Host-Only Ethernet Adapter`:
  - Открыть `Win+R` -> `wf.msc` -> Windows Defender Firewall Properties
  - Для вкладок `Domain Profile`, `Private Profile`, `Public Profile`:
    - Открыть Protected network connections -> Customize
    - Убрать галочку с `VirtualBox Host-Only Ethernet Adapter`
- По окончании выполнения задания убрать проброс портов:
  ```powershell
  netsh interface portproxy delete v4tov4 listenport=8929 listenaddress=192.168.12.1
  netsh interface portproxy delete v4tov4 listenport=5005 listenaddress=192.168.12.1
  netsh interface portproxy delete v4tov4 listenport=2224 listenaddress=192.168.12.1
  ```

## Настройка кластера

По причине того, что локальный docker registry работает по протоколу HTTP,
необходимо добавить его в исключение на каждом из узлов кластера.

- Для каждого из узлов кластера:
  - Создать/изменить файл `/etc/rancher/k3s/registries.yaml`:
    ```yaml
    mirrors:
      "gitlab.local:5005":
        endpoint:
          - "http://gitlab.local:5005"
    configs:
      "gitlab.local:5005":
        tls:
          insecure_skip_verify: true
    ```
  - Перезапустить k3s:
    - Для мастер узла: `systemctl restart k3s`
    - Для воркера: `systemctl restart k3s-agent`
  - В файле `/etc/hosts` добавить строчку с IP-адресом Gitlab: `192.168.12.1 gitlab.local`
- Создать Kubernetes namespace для данной задачи `kubectl create namespace task5`
- В Gitlab для проекта сгенерировать Deploy token `kubernetes` для проекта с правами read_registry (Project -> Settings -> Repository -> Deploy Tokens)
- Добавить Kubernetes secret для docker registry (в docker password подставить токен, полученный на предыдущем шаге):
  ```
  kubectl -n task5 create secret docker-registry todo-regcred --docker-server="gitlab.local:5005" --docker-username=kubernetes --docker-password=<deploy token>
  ```
- Убедиться, что secret был создан: `kubectl -n task5 get secret todo-regcred --output=yaml`

## Деплой PostgreSQL

- Создать Kubernetes ConfigMap с переменными среды для PostgresSQL:
  ```
  kubectl -n task5 create configmap postgres-config --from-literal="POSTGRES_PASSWORD=MYSECRETPASSWORD"
  ```
- В папке с проектом `devops-todo` создать папку `.k8s`
- Создать файл `postgres-deployment.yaml` в папке `.k8s`
  - Описать Kubernetes PersistentVolume для PostgreSQL в `task5` namespace:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: postgres-volume
      namespace: task5
      labels:
        type: local
        app: postgres
    spec:
      storageClassName: manual
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteMany
      hostPath:
        path: "/var/lib/postgres"
      nodeAffinity:
        required:
          nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - ksnode2.local
    ```
  - Описать Kubernetes PersistentVolumeClaim для PostgreSQL в `task5` namespace:
    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: postgres-volume-claim
      namespace: task5
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
    ```
  - Описать Kubernetes Deployment для Postgresql в `task5` namespace:
    ```yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: postgres
      namespace: task5
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: postgres
      template:
        metadata:
          labels:
            app: postgres
        spec:
          containers:
            - name: postgres
              image: postgres:15.4-alpine
              imagePullPolicy: IfNotPresent
              ports:
                - containerPort: 5432
                  protocol: TCP
              envFrom:
                - configMapRef:
                    name: postgres-config
              volumeMounts:
                - mountPath: /var/lib/postgresql/data
                  name: postgres-data
          volumes:
            - name: postgres-data
              persistentVolumeClaim:
                claimName: postgres-volume-claim
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                    - key: kubernetes.io/hostname
                      operator: In
                      values:
                        - ksnode2.local
    ```
  - Описать Kubernetes Service для PostgresSQL в `task5` namespace:
    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres
      namespace: task5
      labels:
        app: postgres
    spec:
      type: NodePort
      ports:
        - port: 5432
      selector:
        app: postgres
    ```
- Применить созданный файл на кластер: `kubectl -n task5 apply -f .k8s/postgres-deployment.yaml`
- Проверить статус пода и убедиться, что он запущен: `kubectl -n task5 get pods`
- Подключиться к PostgreSQL (подставить pod name): `kubectl -n task5 exec --stdin --tty <pod name> -- psql -U postgres`
- Создать базу данных (БД) для приложения `CREATE DATABASE todo;`
- Создать пользователя для приложения `CREATE USER todo WITH ENCRYPTED PASSWORD 'TODOSECRETPASSWORD';`
- **Записать имя пользователя и  пароль. Они понадобятся позднее**
- Сделать пользователя владельцем БД `ALTER DATABASE todo OWNER TO todo;`
- Выдать пользователю все права на БД `GRANT ALL PRIVILEGES ON DATABASE todo TO todo;`
- Отключиться от PostgreSQL

## Деплой приложения

- На хосте отредактировать файл `/etc/hosts` (`C:\Windows\System32\drivers\etc\hosts`) и добавить туда домен `todo.local`, указывающий на мастер узел Kubernetes:
  ```
  192.168.12.10 ksnode1.local todo.local
  192.168.12.20 ksnode2.local
  ```
- Создать Kubernetes ConfigMap с переменными среды для PostgresSQL:
  ```
  kubectl -n task5 create configmap todo-config --from-literal="DATABASE_URL=postgresql://todo:TODOSECRETPASSWORD@postgres:5432/todo?schema=public"
  ```
- Создать файл `deployment.yaml` в папке `.k8s`
  - Описать Kubernetes Deployment для приложения в `task5` namespace:
    - Указать 2 реплики в параметре `replicas`
    - В `spec.template.spec` проиписать `imagePullSecrets`, чтобы Kubernetes знал какие учетные данные использовать для подключения к приватному docker registry:
      ```yaml
      imagePullSecrets:
        - name: todo-regcred
      ```
    - В качестве образа использовать `gitlab.local:5005/<username>/devops-todo:main` (username подставить из Gitlab)
    - В качестве порта указать порт `3000`
    - В параметре `imagePullPolicy` указать `Always`, чтобы Kubernetes скачивал свежий образ приложения при обновлении
    - В `configMapRef` подставить ConfigMap приложения, созданный на предыдущих шагах (`todo-config`)
    - Параметры `volumeMounts`, `volumes` и `affinity` для деплоя приложения не нужны
    - Для запуска миграций в `spec.template.spec` описать `initContainers`:
      ```yaml
      initContainers:
        - name: init-migrate
          image: gitlab.local:5005/dmitry.sinelnikov/devops-todo:main
          imagePullPolicy: Always
          command: ['sh', '-c', 'npm run migrate deploy']
          envFrom:
            - configMapRef:
                name: todo-config
      ```
  - Описать Kubernetes Service для приложения в `task5` namespace:
    - Использовать порт `3000`
  - Описать Kubernetes Ingress для приложения в `task5` namespace:
    ```yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: todo
      annotations:
        traefik.ingress.kubernetes.io/router.entrypoints: web
      labels:
        app: todo
    spec:
      ingressClassName: traefik
      rules:
      - host: todo.local
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: todo
                port:
                  number: 3000
    ```
- Применить созданный файл на кластер: `kubectl -n task5 apply -f .k8s/deployment.yaml`
- Проверить статус подов и убедиться, что они запущены: `kubectl -n task5 get pods`
- Проанализировать распределение подов по узлам: `kubectl -n task5 get pods -o wide`
- В браузере открыть ссылку http://todo.local и убедиться, что приложение работает

## Деплой приложение при помощи Gitlab CI

### Подготовка

- В файл `/etc/hosts` на сервере Gitlab Добавить домены узлов кластера:
  ```
  192.168.12.10 ksnode1.local
  192.168.12.20 ksnode2.local
  ```
- На странице проекта в GitLab открыть Settings -> CI/CD -> Variables
- Добавить переменную `KUBE_CONFIG` с типом File и вставить в нее конфиг Kubernetes (`~/.kube/config` с локальной машины)

### Редактирование Gitlab CI

- В файле `.gitlab-ci.yml` заменить задачу `deploy` на новую:
  - В качестве образа использовать `bitnami/kubectl:1.27` с пустым entrypoint:
    ```yaml
    image:
      name: bitnami/kubectl:1.27
      entrypoint: [""]
    ```
  - В начале выполнения задачи установить `~/.kube/config` из переменной Gitlab:
    ```bash
    mkdir -p ~/.kube
    cp "$KUBE_CONFIG" ~/.kube/config
    chmod 600 ~/.kube/config
    ```
  - При выполнении задачи применить файл `.k8s/deployment.yaml` на кластере и запустить обновление подов:
    ```bash
    kubectl apply -f .k8s/deployment.yaml
    kubectl -n task5 rollout restart deployment/todo
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
        <h1>Hello world!</h1>
        <TodoList />
      </main>
    )
  }
  ```
- Запушить изменения в ветку `main`
- Дождаться успешного завершения pipeline-а
- Проверить состояние подов: `kubectl -n task5 get pods -o wide`
- Проверить внесенные в приложение изменения: http://todo.local