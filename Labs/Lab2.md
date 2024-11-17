# GitLab

## Установка Gitlab

### Native

[Официальная документация](https://about.gitlab.com/install/#debian)

### Docker

[Официальная документация](https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-engine)

### Docker compose (рекомендуемый способ)

[Официальная документация](https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-compose)

```yaml
version: "3"

services:
  gitlab:
    image: gitlab/gitlab-ce
    container_name: gitlab
    hostname: localhost
    shm_size: 256m
    restart: unless-stopped
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    ports:
      - "8929:8929"
      - "2224:22"
```

## Настройка Gitlab

[Официальная документация](https://docs.gitlab.com/ee/install/docker.html#pre-configure-docker-container)

- Дождаться запуска Gitlab по логам (`docker compose logs -f gitlab`)
- Открыть локальный Gitlab (http://localhost:8929)
- Убедиться, что отображается страница входа
- Получить пароль пользователя `root` (`docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password`)
- Зайти в Gitlab под пользователем `root` с использованием полученного пароля
- Сменить пароль root пользователя (http://localhost:8929/-/profile/password/edit)
- Открыть администрирование пользователей (http://localhost:8929/admin/users)
- Создать своего пользователя с правами администратора
- Перезайти под новым пользователем

## Установка Gitlab Runner

### Native

[Официальная документация](https://docs.gitlab.com/runner/install/linux-repository.html#installing-gitlab-runner)

### Docker

[Официальная документация](https://docs.gitlab.com/runner/install/docker.html#option-1-use-local-system-volume-mounts-to-start-the-runner-container)

### Docker compose (рекомендуемый способ)

```yaml
version: "3"

services:
  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    container_name: gitlab-runner
    volumes:
      - ./runner:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
```

## Регистрация Gitlab Runner

[Официальная документация](https://docs.gitlab.com/runner/register/index.html)

- Открыть страницу управления runner-ами (http://localhost:8929/admin/runners)
- Создать новый runner (в качестве тега указать `shared`)
- Запустить регистрацию runnner-а (`docker compose run --rm runner register`)
- Ввести адрес Gitlab во внутренней сети docker (`http://gitlab:8929`)
- Ввести полученный в Gitlab токен
- Executor - `docker`
- Default Docker image - `docker:stable`
- Запустить runner (`docker compose up -d runner`)
- Открыть страницу управления runner-ами (http://localhost:8929/admin/runners)
- Убедиться, что runner запущен