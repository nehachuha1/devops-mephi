# Kubernetes

## Подготовка

### Требования

- Две виртуальные машины (2 ядра, 2ГБ памяти) c Debian 12
- Сеть со статическими адресами между машинами
- Предустановленный docker

### Пример настройки для VirtualBox

- Создать виртуальную машину (2 ядра, 2ГБ памяти) с именем `ksnode1`
- Установить Debian 12
- В VirtualBox cоздать новую Host-only сеть (Инструменты -> Виртуальные сети хоста):
  - IPv4 адрес `192.168.12.1`
  - DHCP сервер выключен
- В настройках виртуальной машины в Сети включить Адаптер 2:
  - Тип подключения: Виртуальный адаптер хоста
  - Имя: выбрать созданную выше Host-only сеть
- В виртуальной машине Посмотреть названия интерфейсов командой `ip addr` (названия интерфейсов скорее всего будут начинаться с `enp0`). Пример: `enp0s3`, `enp0s8`
- В виртуальной машине отредактировать файл `/etc/network/interfaces`:
  - Для первого интерфейса (`enp0s3`) прописать динамическое получение адреса:
    ```
    auto enp0s3
    allow-hotplug enp0s3
    iface enp0s3 inet dhcp
    ```
  - Для второго интерфейса (`enp0s8`) прописать статический адрес `192.168.12.10`
    ```
    auto enp0s8
    allow-hotplug enp0s8
    iface enp0s8 inet static
    network 192.168.12.0
    address 192.168.12.10
    netmask 255.255.255.0
    broadcast 192.168.12.255
    ```
- Установить имя хоста `ksnode1` (`hostnamectl hostname ksnode1.local`)
- Установить пакет `ssh` (`apt install ssh`)
- Перезагрузить виртуальную машину (`reboot`)
- Подключиться к машине через ssh (`ssh 192.168.12.10`)
- Установить docker
- Установить дополнительные пакеты (редактор, curl, wget)
- Выключить виртуальную машину
- В VirtualBox склонировать машину:
  - Имя: `ksnode2`
  - Политика MAC-адреса: Сгенерировать новые MAC-адреса всех сетевых адаптеров
- Запустить виртуальную машину `ksnode2`
- В виртуальной машине `ksnode2` отредактировать файл `/etc/network/interfaces`:
  - Для второго интерфейса (`enp0s8`) изменить адрес на `192.168.12.20`:
  ```
  auto enp0s8
  allow-hotplug enp0s8
  iface enp0s8 inet static
  network 192.168.12.0
  address 192.168.12.20
  netmask 255.255.255.0
  broadcast 192.168.12.255
  ```
- Установить имя хоста `ksnode2` (`hostnamectl hostname ksnode2.local`)
- Остановить виртуальную машину `ksnode2`
- Прописать доменные имена виртуальных машин в файле `/etc/hosts` (`C:\Windows\System32\drivers\etc\hosts`) хоста:
  ```
  192.168.12.10 ksnode1.local
  192.168.12.20 ksnode2.local
  ```
- Запустить обе виртуальные машины
- Подключиться к обоим машинам по ssh по доменному имени (`ssh ksnode1.local`, `ssh ksnode2.local`)
- На всех виртуальных машинах в конец файла `/etc/hosts` добавить:
  ```
  192.168.12.10 ksnode1.local
  192.168.12.20 ksnode2.local
  ```
- Проверить доступность сети на вирутальных машинах:
  - `ping ksnode1.local`
  - `ping ksnode2.local`

## Установка K3S

В данном задании предполагается, что кластер Kubernetes будет состоять из двух узлов: сервера и агента.

`ksnode1` будет являться сервером, а `ksnode2` - агентом.

### Установка сервера

- На машине `ksnode1` Установить k3s:
  - `enp0s8` заменить на второй интерфейс виртуальной машины

  ```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=192.168.12.10 --flannel-iface=enp0s8" sh -
  ```

- Получить токен для подключения агетов: `cat /var/lib/rancher/k3s/server/node-token`

### Установка и подключение агента

- На машине `ksnode2` установить k3s и подключиться к серверу `ksnode1`:
  - В K3S_TOKEN подставить токен, полученный после установки сервера
  - `enp0s8` заменить на второй интерфейс виртуальной машины
 командой
  ```bash
  curl -sfL https://get.k3s.io | K3S_URL=https://ksnode1.local:6443 K3S_TOKEN=mynodetoken INSTALL_K3S_EXEC="--node-ip=192.168.12.20 --flannel-iface=enp0s8" sh -
  ```

### Проверка кластера

- Запустить команду `kubectl get nodes -o wide` на обоих узлах
- Проанализировать результаты (в выводе команды должно быть два узла)

## Настройка доступа к кластеру Kubernetes

- Установить `kubectl` на свою машину:
  - [Инструкция для Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)
  - [Инструкция для macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/#install-kubectl-on-macos)
  - [Инструкция для Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-on-windows)
- Скопировать файл `/etc/rancher/k3s/k3s.yaml` с `ksnode1` на `ksnode2` на свою машину по пути `~/.kube/config`
- Для Linux, macOS установить права на файл `~/.kube/config`: `chmod 600 ~/.kube/config`
- В файле `~/.kube/config` в строчке `server: https://127.0.0.1:6443` заменить адрес на `https://ksnode1.local:6443`
- Проверить доступ к кластеру командой `kubectl get nodes`

## Деплой Kubernetes Dashboard

- Установить `helm` на свою машину ([инструкция](https://helm.sh/docs/intro/install/#through-package-managers))
- Добавить helm репозиторий Kuberetes Dashboard:
  ```bash
  helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
  ```
- Запустить деплой Kuberenes Dashboard:
  ```
  helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
  ```
- Проверить статус пода: `kubectl get pods --namespace kubernetes-dashboard`
- Создать пользователя с правами администратора для kubernetes-dashboard:
  - Создать файл `dashboard-admin.yaml`:
    ```yaml
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: admin-user
        namespace: kubernetes-dashboard
    ```
  - Применить конфигурацию на кластер: `kubectl apply -f dashboard-admin.yaml`
  - Получить токен, созданного пользователя: `kubectl -n kubernetes-dashaboard create token admin-user`
  - Пробросить порт `kubernetes-dashboard` из кластера на локальную машину: `kubectl -n kubernetes-dashboard port-forward service/kubernetes-dashboard-kong-proxy 8443:443`
  - Открыть проброшенный порт в браузере: https://localhost:8443
  - Ввести токен, полученный на предыдущих шагах
  - Проанализировать данные, отображающиеся в `kubernetes-dashboard`