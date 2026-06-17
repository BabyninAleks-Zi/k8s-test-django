# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8080/admin/](http://127.0.0.1:8080/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Деплой в облако

Этот README описывает локальную разработку и общие настройки приложения.

Актуальные инструкции по деплою в dev окружение Yandex Cloud, ссылки на
выделенные облачные ресурсы и адрес работающей версии сайта находятся в
отдельном документе:

[`deploy/environments/yc-sirius-dev/DEPLOY.md`](deploy/environments/yc-sirius-dev/DEPLOY.md)

## Как развернуть сайт в Minikube на MacOs

Для локального Kubernetes-развёртывания понадобятся Docker Desktop, Minikube, kubectl и Helm.

Запустите Minikube:

```shell
$ minikube start --driver=docker
$ kubectl get nodes
```

Соберите образ сайта внутри Minikube:

```shell
$ minikube image build -t django_app:latest ./backend_main_django
$ minikube image ls
```

### Как развернуть PostgreSQL в кластере

База данных разворачивается в Kubernetes через Helm chart Bitnami PostgreSQL:

```shell
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
$ helm install postgresql bitnami/postgresql \
    --set auth.username=test_k8s \
    --set auth.password='<database-password>' \
    --set auth.database=test_k8s \
    --set auth.postgresPassword='<postgres-password>'
```

Проверьте, что база запущена:

```shell
$ kubectl get pods
$ kubectl get svc
```

В кластере должен появиться pod `postgresql-0` и сервис `postgresql` на порту `5432`.

Подключиться к базе можно временным pod с `psql`:

```shell
$ kubectl run psql-client \
    --rm \
    -it \
    --restart=Never \
    --image=postgres:12.0-alpine \
    --env=PGPASSWORD='<database-password>' \
    -- psql -h postgresql -U test_k8s -d test_k8s
```

### Как создать Secret с настройками Django

Секретные настройки не хранятся в репозитории. Файл `kubernetes/django-secret.yaml` добавлен в `.gitignore`.

Создайте локальный манифест Secret:

```shell
$ kubectl create secret generic django \
    --from-literal=SECRET_KEY='<django-secret-key>' \
    --from-literal=DEBUG='FALSE' \
    --from-literal=ALLOWED_HOSTS='star-burger.test,127.0.0.1,localhost' \
    --from-literal=DATABASE_URL='postgres://test_k8s:<database-password>@postgresql:5432/test_k8s' \
    --dry-run=client \
    -o yaml > kubernetes/django-secret.yaml
```

Примените Secret:

```shell
$ kubectl apply -f kubernetes/django-secret.yaml
$ kubectl get secret django
```

Если пароль PostgreSQL изменился, обновите значение `DATABASE_URL` в `kubernetes/django-secret.yaml` и снова примените Secret:

```shell
$ kubectl apply -f kubernetes/django-secret.yaml
$ kubectl rollout restart deployment django-site
```

### Как применить манифесты сайта

Примените Kubernetes-манифесты:

```shell
$ kubectl apply -f kubernetes/
```

Проверьте состояние ресурсов:

```shell
$ kubectl get pods
$ kubectl get svc
$ kubectl get ingress
$ kubectl get cronjobs
```

Сервис `django` должен иметь тип `ClusterIP`, а сайт должен запускаться через Deployment `django-site`.

### Как применить миграции

Миграции запускаются отдельной Kubernetes Job:

```shell
$ kubectl delete job django-migrate --ignore-not-found
$ kubectl apply -f kubernetes/django-migrate-job.yaml
$ kubectl logs job/django-migrate
```

Успешный запуск выглядит так:

```text
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  No migrations to apply.
```

Если база новая, создайте суперпользователя:

```shell
$ kubectl exec -it deploy/django-site -- ./manage.py createsuperuser
```

### Как открыть сайт через Ingress

Включите Ingress controller:

```shell
$ minikube addons enable ingress
$ kubectl get pods -n ingress-nginx
```

Добавьте локальный домен в `/etc/hosts`:

```shell
$ echo "127.0.0.1 star-burger.test" | sudo tee -a /etc/hosts
```

На macOS с Docker driver запустите tunnel в отдельном терминале и оставьте его работать:

```shell
$ sudo minikube tunnel
```

Откройте сайт:

```text
http://star-burger.test/admin/
```

Если Django отвечает `400 Bad Request`, проверьте, что в Secret значение `ALLOWED_HOSTS` содержит `star-burger.test`, а `DEBUG` равен `FALSE`:

```shell
$ kubectl exec deploy/django-site -- printenv ALLOWED_HOSTS DEBUG
```

### Как проверить очистку сессий

Регулярная очистка устаревших Django-сессий настроена через CronJob `django-clearsessions`.

Проверить CronJob:

```shell
$ kubectl get cronjobs
```

Запустить задачу вручную:

```shell
$ kubectl delete job django-clearsessions-once --ignore-not-found
$ kubectl create job django-clearsessions-once --from=cronjob/django-clearsessions
$ kubectl get jobs
$ kubectl logs job/django-clearsessions-once
```

Команда `./manage.py clearsessions` удаляет только просроченные Django-сессии.
