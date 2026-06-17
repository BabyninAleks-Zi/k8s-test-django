# YC Sirius Dev Deploy

## Рабочая версия и ресурсы

Сайт в dev окружении YC Sirius доступен по адресу:

[https://edu-aleksandr-babynin.yc-sirius-dev.pelid.team/](https://edu-aleksandr-babynin.yc-sirius-dev.pelid.team/)

Облачная инфраструктура:

- Cloud: `sirius-cloud`
- Folder: `dev`
- Kubernetes cluster: `yc-sirius-dev`
- Namespace: `edu-aleksandr-babynin`
- [Кластер в Yandex Cloud Console](https://console.cloud.yandex.ru/folders/b1gtcctl0mkamhmvoq79/managed-kubernetes/cluster/cato1chddk98dhsru9k3)

## Что нужно сайту для работы

Окружение `yc-sirius-dev` создаётся и настраивается отдельно от приложения.
В процессе обычного деплоя сайт не создаёт Kubernetes-кластер, домен, TLS,
Application Load Balancer или Managed PostgreSQL.

Приложению нужны:

- Kubernetes namespace с доступом на создание `Deployment`, `Service`, `Job`,
  `CronJob`, `ConfigMap` и `Secret`.
- Docker image приложения, опубликованный в Docker Hub.
- Managed PostgreSQL с готовой базой данных и пользователем.
- SSL-сертификат PostgreSQL, смонтированный в контейнер как
  `/etc/postgresql/root.crt`.
- Домен вида `edu-your-namespace.yc-sirius-dev.pelid.team`, уже направленный
  через ALB на `main-nginx`.

Настройки приложения передаются через Secret `django`:

| Переменная | Назначение |
| --- | --- |
| `SECRET_KEY` | Секретный ключ Django. |
| `DEBUG` | `FALSE` для серверного окружения. |
| `ALLOWED_HOSTS` | Домены и host-ы, с которых Django принимает запросы. |
| `DATABASE_URL` | DSN подключения к PostgreSQL. |

## Как собрать и опубликовать Docker image

В dev окружении образ версионируется hash-ем git-коммита.

Перед сборкой задайте переменные:

```shell
DOCKERHUB_USERNAME="<your-dockerhub-username>"
IMAGE_NAME="k8s-test-django"
IMAGE="$DOCKERHUB_USERNAME/$IMAGE_NAME"
COMMIT_HASH="$(git log -1 --format=%H)"
```

В Docker Hub заранее создайте repository с таким же именем, как `IMAGE_NAME`,
или замените `IMAGE_NAME` на имя уже созданного repository.

Проверьте tag, который будет опубликован:

```shell
echo "$IMAGE:$COMMIT_HASH"
```

Соберите образ Django Site:

```shell
docker build \
  --tag "$IMAGE:$COMMIT_HASH" \
  ./backend_main_django
```

Авторизуйтесь в Docker Hub:

```shell
docker login
```

Опубликуйте образ:

```shell
docker push "$IMAGE:$COMMIT_HASH"
```

Проверьте, что образ можно скачать из Docker Hub:

```shell
docker pull "$IMAGE:$COMMIT_HASH"
```

Пример опубликованного dev-образа:

```text
aleksandrbabynin/k8s-test-django:ff98ae134ac7494780c2030e27cc2ed4832a3a5c
```

Чтобы опубликовать образ для старого коммита, соберите код из checkout-а этого
коммита и используйте тот же принцип тегирования:

```shell
COMMIT_HASH="$(git log -1 --format=%H)"
docker build --tag "$IMAGE:$COMMIT_HASH" ./backend_main_django
docker push "$IMAGE:$COMMIT_HASH"
```

Старые образы не затираются, потому что каждый commit hash даёт отдельный
Docker tag.

## Как задеплоить код

Dev окружение в Yandex Cloud уже подготовлено Девманом. Для текущего
окружения используйте Ваш namespace, который Вам выдали в Yandex Cloud и примените его везде где увидите ниже your-namespace:

```shell
edu-your-namespace
```

Перед деплоем проверьте, что `kubectl` подключён к нужному кластеру:

```shell
kubectl config current-context
```

Ожидаемый контекст:

```text
yc-sirius-dev
```

Рабочие манифесты окружения лежат в каталоге
`deploy/environments/yc-sirius-dev/kubernetes/`.

### Задеплоить Django Site

Убедитесь, что в манифесте
`deploy/environments/yc-sirius-dev/kubernetes/django-deployment.yaml` указан
нужный Docker image:

Пример:

```text
aleksandrbabynin/k8s-test-django:ff98ae134ac7494780c2030e27cc2ed4832a3a5c
```

Примените Service, Deployment и CronJob:

```shell
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/django-service.yaml
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/django-deployment.yaml
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/django-clearsessions-cronjob.yaml
```

Проверьте rollout:

```shell
kubectl rollout status deployment/django-site -n edu-your-namespace
kubectl get pods,svc,deploy,cronjob -n edu-your-namespace
```

Примените миграции:

```shell
kubectl delete job django-migrate -n edu-your-namespace --ignore-not-found
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/django-migrate-job.yaml
kubectl wait --for=condition=complete job/django-migrate -n edu-your-namespace --timeout=180s
kubectl logs job/django-migrate -n edu-your-namespace
```

Проверьте Django напрямую через port-forward:

```shell
kubectl port-forward service/django 8082:80 -n edu-your-namespace
```

Откройте в браузере:

```text
http://127.0.0.1:8082/admin/
```

Создайте суперпользователя вручную, чтобы пароль не попадал в git, shell history
или логи:

```shell
kubectl exec -it deployment/django-site -n edu-your-namespace -- ./manage.py createsuperuser
```

### Переключить Nginx на Django

В новом кластере `yc-sirius-dev` публичный трафик идёт по цепочке:

```text
Browser -> ALB -> main-nginx -> django
```

Поэтому `main-nginx` должен работать как reverse proxy на Kubernetes Service
`django`.

Примените ConfigMap Nginx:

```shell
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/main-nginx-configmap.yaml
```

Перезапустите Nginx, чтобы Pod перечитал ConfigMap:

```shell
kubectl rollout restart deployment/main-nginx -n edu-your-namespace
kubectl rollout status deployment/main-nginx -n edu-your-namespace
```

Проверьте ресурсы в namespace:

```shell
kubectl get pods,svc,deploy,configmap,secret -n edu-your-namespace
```

Проверьте сайт по HTTPS:

```shell
curl -I https://edu-your-namespace.yc-sirius-dev.pelid.team/
```

Если открыть сайт по HTTP, Application Load Balancer должен перенаправить
запрос на HTTPS:

```shell
curl -I http://edu-your-namespace.yc-sirius-dev.pelid.team/
```

Свежие HTTP-запросы можно увидеть в логах Nginx:

```shell
kubectl logs deployment/main-nginx -n edu-your-namespace --tail=30
kubectl logs deployment/django-site -n edu-your-namespace --tail=30
```

### Запустить management-команду

Разовую management-команду удобно запускать в уже работающем Django Pod:

```shell
kubectl exec -it deployment/django-site -n edu-your-namespace -- ./manage.py check
```

Примеры:

```shell
kubectl exec -it deployment/django-site -n edu-your-namespace -- ./manage.py createsuperuser
kubectl exec -it deployment/django-site -n edu-your-namespace -- ./manage.py shell
```

Миграции запускайте через отдельный `Job`, чтобы результат был виден в статусе
Kubernetes:

```shell
kubectl delete job django-migrate -n edu-your-namespace --ignore-not-found
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/django-migrate-job.yaml
kubectl logs job/django-migrate -n edu-your-namespace
```

### Где искать трейсбеки

Сначала смотрите логи приложения:

```shell
kubectl logs deployment/django-site -n edu-your-namespace --tail=100
```

Если Pod не стартует или перезапускается, смотрите описание Pod:

```shell
kubectl get pods -n edu-your-namespace
kubectl describe pod <pod-name> -n edu-your-namespace
```

Если сайт не открывается снаружи, но Django Pod живой, проверьте Nginx:

```shell
kubectl logs deployment/main-nginx -n edu-your-namespace --tail=100
kubectl describe deployment main-nginx -n edu-your-namespace
```

## Однократная подготовка dev окружения

Namespace:

```shell
edu-your-namespace
```

Проверьте доступ к namespace:

```shell
kubectl get secrets -n edu-your-namespace
```

В окружении уже должен быть Secret `postgres` с параметрами подключения к
Managed PostgreSQL.

### Создать Secret с настройками Django

Secret `django` хранит переменные окружения приложения:

- `SECRET_KEY`
- `DEBUG`
- `ALLOWED_HOSTS`
- `DATABASE_URL`

Создайте локальный временный файл с настройками. Значение `DATABASE_URL`
возьмите из документа с ресурсами или из выданного секрета `postgres`; в конце
DSN должны быть параметры SSL:

```text
sslmode=verify-full&sslrootcert=/etc/postgresql/root.crt&target_session_attrs=read-write
```

```shell
NAMESPACE="edu-your-namespace"
DOMAIN="$NAMESPACE.yc-sirius-dev.pelid.team"

cat > /tmp/django.env <<EOF
SECRET_KEY=$(openssl rand -base64 48 | tr -d '\n')
DEBUG=FALSE
ALLOWED_HOSTS=127.0.0.1,localhost,$DOMAIN
DATABASE_URL=postgres://user:password@host:6432/dbname?sslmode=verify-full&sslrootcert=/etc/postgresql/root.crt&target_session_attrs=read-write
EOF

kubectl create secret generic django \
  -n "$NAMESPACE" \
  --from-env-file=/tmp/django.env \
  --dry-run=client -o yaml \
  | kubectl apply -f -

rm /tmp/django.env
```

Проверьте, что Secret появился:

```shell
kubectl get secret django -n edu-your-namespace
```

Секреты, пароли и DSN не сохраняем в git.

### Создать Secret с SSL-сертификатом PostgreSQL

Сертификат для проверки TLS-подключения хранится в ключе `root.crt` секрета
`postgres`. Для монтирования сертификата отдельным файлом создаём отдельный
Secret `postgres-root-cert`:

```shell
kubectl get secret postgres -n edu-your-namespace -o json \
  | jq '{
      apiVersion: "v1",
      kind: "Secret",
      metadata: {
        name: "postgres-root-cert",
        namespace: "edu-your-namespace"
      },
      type: "Opaque",
      data: {
        "root.crt": .data["root.crt"]
      }
    }' \
  | kubectl apply -f -
```

Проверьте, что Secret появился:

```shell
kubectl get secret postgres-root-cert -n edu-your-namespace
```

Секреты, пароли, DSN и содержимое сертификата не сохраняем в git.

### Проверить автоматическое монтирование сертификата

Создайте тестовый Pod:

```shell
kubectl apply -f deploy/environments/yc-sirius-dev/kubernetes/psql-client-pod.yaml
kubectl wait pod/psql-client -n edu-your-namespace --for=condition=Ready --timeout=120s
```

Проверьте, что файл сертификата появился внутри контейнера:

```shell
kubectl exec -n edu-your-namespace psql-client -- ls -l /root/.postgresql/root.crt
```

Установите `psql` внутри тестового Pod:

```shell
kubectl exec -it -n edu-your-namespace psql-client -- bash
apt-get update
apt-get install -y postgresql-client ca-certificates
```

Проверьте подключение к PostgreSQL:

```shell
PGPASSWORD="$password" psql "host=$host port=$port dbname=$name user=$username sslmode=verify-full sslrootcert=/root/.postgresql/root.crt target_session_attrs=read-write"
```

Внутри `psql`:

```sql
SELECT current_database(), current_user, version();
\q
```

После проверки удалите тестовый Pod:

```shell
kubectl delete pod psql-client -n edu-your-namespace
```

Смена SSL-сертификата не требует пересборки Docker-образа: достаточно обновить
Secret `postgres-root-cert` и пересоздать Pod, который монтирует этот Secret.
