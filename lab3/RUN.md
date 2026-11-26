# Запуск лабораторной № 3

[Отчёт и результаты проверки](README.md). Здесь собраны команды для повторного запуска.

## Развёртывание

Команды выполняются из папки `containers_orchestration/lab3`.

### 1. Подготовка Kubernetes

Нужны работающий Docker, [kubectl](https://kubernetes.io/docs/tasks/tools/) и [Minikube](https://minikube.sigs.k8s.io/docs/start/). Если локальный кластер уже настроен, установка Minikube не обязательна.

```bash
docker version
minikube start -p lab3 --driver=docker --cpus=2 --memory=4096
kubectl config use-context lab3
kubectl get nodes
```

Перед применением файлов проверить, что выбран именно учебный кластер:

```bash
kubectl config current-context
```

### 2. PostgreSQL

Сначала создать namespace, затем конфигурацию, Secret, Service и Deployment базы:

```bash
kubectl apply -f namespace.yml
kubectl apply -f pg_configmap.yml -f pg_secret.yml -f pg_service.yml -f pg_deployment.yml
kubectl -n lab3 rollout status deployment/postgres --timeout=180s
kubectl -n lab3 get pods,services
kubectl -n lab3 describe secret postgres-secret
```

Проверка `pg_isready` обращается к `127.0.0.1:5432`. Pod станет готовым после запуска основного TCP-сервера PostgreSQL, а не временного сервера первоначальной настройки. `describe secret` показывает названия полей и размеры значений без вывода паролей.

### 3. Nextcloud

Запустить Nextcloud после готовности базы, чтобы автоматическая установка могла создать таблицы:

```bash
kubectl apply -f nextcloud_configmap.yml -f nextcloud_secret.yml -f nextcloud.yml -f nextcloud_service.yml
kubectl -n lab3 rollout status deployment/nextcloud --timeout=600s
kubectl -n lab3 logs deployment/nextcloud --tail=60
kubectl -n lab3 get deployments,pods,services,configmaps,secrets
kubectl -n lab3 describe deployment nextcloud
```

Ожидается по одной готовой реплике у обоих Deployment и состояние `1/1 Running` у обоих Pod. Первая загрузка образа и установка Nextcloud могут занять несколько минут. При ошибке смотреть `kubectl -n lab3 describe pod <имя-pod>` и логи нужного Deployment.

### 4. Веб-интерфейс

В отдельном терминале выполнить и оставить работающей команду:

```bash
kubectl -n lab3 port-forward service/nextcloud 8080:80
```

Если при загрузке CSS/JS появляются сбросы соединений, а `kubectl` пишет `error creating error stream ... Timeout occurred`, остановить туннель и запустить его в Bash с другим поддерживаемым транспортом:

```bash
KUBECTL_PORT_FORWARD_WEBSOCKETS=false kubectl -n lab3 port-forward service/nextcloud 8080:80
```

Этот вариант использовался при проверке на временном k3s-кластере. Он меняет транспорт клиентского соединения. Переменная описана в [заметке Kubernetes о переходе на WebSocket](https://kubernetes.io/blog/2024/08/20/websockets-transition/).

Открыть [http://127.0.0.1:8080](http://127.0.0.1:8080). Логин: `lab3admin`. Пароль указан в `nextcloud_secret.yml` в поле `NEXTCLOUD_ADMIN_PASSWORD`.

После входа открыть раздел файлов. Дополнительно проверить статус из другого терминала:

```bash
curl --fail http://127.0.0.1:8080/status.php
```

В JSON ожидаются `installed: true`, `maintenance: false` и `needsDbUpgrade: false`. Один только код HTTP 200 от `status.php` не подтверждает все эти условия.

Вариант из методички:

```bash
minikube -p lab3 service nextcloud -n lab3 --url
```

Если команда выдаёт адрес ноды вместо localhost, этот IP должен быть в `NEXTCLOUD_TRUSTED_DOMAINS`. Для такого способа доступа ещё до шага 3 получить IP командой `minikube -p lab3 ip` и добавить его в `nextcloud_configmap.yml`. Уже установленный стенд проще открыть через основной способ `port-forward`, который работает с заданными `localhost` и `127.0.0.1`. Перезапускать только Deployment Nextcloud ради смены переменных здесь не следует: его `emptyDir` будет удалён, а прежние таблицы останутся в базе.

### 5. Проверки состояния Nextcloud

| Проверка | Условие | Действие Kubernetes при сбое |
| --- | --- | --- |
| Startup | TCP-порт 80 открылся; на запуск выделено до 600 секунд | После исчерпания попыток перезапускает контейнер |
| Liveness | TCP-порт 80 доступен | После трёх неудач подряд перезапускает контейнер |
| Readiness | Страница `/index.php/login` отвечает успешно с заголовком `Host: localhost` | После трёх неудач подряд исключает Pod из готовых адресов Service |

Startup откладывает остальные пробы до завершения запуска. TCP Liveness проверяет доступность веб-сервера, но не состояние БД или PHP: сбой PostgreSQL не должен перезапускать Nextcloud и мешать опыту ниже. Readiness обращается к странице входа и проверяет, может ли приложение её обработать. Правила работы проб описаны в [документации Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/probes/).

Проверка конфигурации проб и состояния Pod:

```bash
kubectl -n lab3 describe pod -l app=nextcloud
kubectl -n lab3 get endpointslices -l kubernetes.io/service-name=nextcloud
```

## Опыт с пересозданием PostgreSQL

Опыт удаляет учебную базу. Выполнять его после входа в Nextcloud и сохранения скриншотов рабочего приложения.

Сначала посмотреть существующие таблицы и Pod:

```bash
kubectl -n lab3 exec deployment/postgres -- psql -U nextcloud -d nextcloud -c '\dt'
kubectl -n lab3 get pods
```

Удалить реплику PostgreSQL, дождаться удаления Pod, затем вернуть одну реплику:

```bash
kubectl -n lab3 scale deployment/postgres --replicas=0
kubectl -n lab3 wait --for=delete pod -l app=postgres --timeout=120s
kubectl -n lab3 scale deployment/postgres --replicas=1
kubectl -n lab3 rollout status deployment/postgres --timeout=180s
kubectl -n lab3 exec deployment/postgres -- psql -U nextcloud -d nextcloud -c '\dt'
kubectl -n lab3 wait --for=condition=Ready=false pod -l app=nextcloud --timeout=90s
kubectl -n lab3 get pods
```

Ожидается отсутствие таблиц в новой БД и `0/1 Running` у Nextcloud. В методичке в вопросе встречается имя `postgres-deployment`, но в её YAML и в этой работе Deployment называется `postgres`.

Чтобы посмотреть ответ Nextcloud, остановить прежний `port-forward` сочетанием Ctrl+C и направить порт непосредственно к Pod через Deployment:

```bash
kubectl -n lab3 port-forward deployment/nextcloud 8080:80
```

В другом терминале:

```bash
curl -i http://127.0.0.1:8080/index.php/login
kubectl -n lab3 logs deployment/nextcloud --tail=30
```

Для новой чистой установки после опыта удалить оба Deployment и создать их в прежнем порядке. Это сброс стенда, а не восстановление потерянных данных:

```bash
kubectl -n lab3 delete deployment nextcloud postgres
kubectl -n lab3 wait --for=delete pod -l app=nextcloud --timeout=120s
kubectl -n lab3 wait --for=delete pod -l app=postgres --timeout=120s
kubectl apply -f pg_deployment.yml
kubectl -n lab3 rollout status deployment/postgres --timeout=180s
kubectl apply -f nextcloud.yml
kubectl -n lab3 rollout status deployment/nextcloud --timeout=600s
```

## Удаление стенда

Удалить только ресурсы этой лабораторной, включая все временные данные:

```bash
kubectl delete namespace lab3
```

Если для лабораторной создавался отдельный профиль Minikube, его также можно удалить:

```bash
minikube delete -p lab3
```

## Примечания к манифестам

`ALLOW_EMPTY_PASSWORD` перенесена из примера в ConfigMap со значением `no`. Официальный образ `nextcloud` не использует эту переменную; непустой пароль задаётся через `NEXTCLOUD_ADMIN_PASSWORD`. `NEXTCLOUD_UPDATE=1` сохранена из задания. Описание поддерживаемых переменных: [документация образа Nextcloud](https://github.com/nextcloud/docker#auto-configuration-via-environment-variables).

У PostgreSQL Service типа ClusterIP, поскольку к нему обращается только Nextcloud внутри кластера. Для Nextcloud предусмотрен NodePort, как в методичке. Лимит памяти Nextcloud увеличен до 1 GiB, чтобы оставить запас для первой установки и PHP.
